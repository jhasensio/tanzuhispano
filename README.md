# Tanzuhispano

Se trata de crear una topología que combine la funcionalidad Multicluster de ANTREA con el GeoBalanceo de carga y los servicios de Ingress (AVI). La topología buscada esta represntada en la siguiente figura.

<img width="1088" alt="image" src="https://github.com/jhasensio/tanzuhispano/assets/37454410/3caaa3d5-cdbe-4f73-94bb-c47a0ec95612">


El primer paso sería crear la infraestructura de kubernetes compuestas por 3 cluster de 1 Control Plane y 3 Miembros en mi caso.  En este caso he usado clusters vanilla 1.24 pero cualquier versión sería adecuada. Es importante utilizar redes de Pod y de Services NO SOLAPADAS. En este caso una manera de inicializar los clusters para respetar el direccionamiento indicado en el dibujo de arriba sería la que se muestra a continuación

Saltar a VM designada como control-plane para el cluster TANZUHISPANO-01
 
```
      kubeadm init --service-cidr=10.228.96.0/22 --pod-network-cidr=10.118.96.0/22
```

Saltar a VM designada como control-plane para el cluster TANZUHISPANO-DB
 
```
      kubeadm init --service-cidr=10.228.112.0/22 --pod-network-cidr=10.118.112.0/22
```

 Saltar a VM designada como control-plane para el cluster TANZUHISPANO-02
 
```
      kubeadm init --service-cidr=10.228.128.0/22 --pod-network-cidr=10.118.128.0/22
```

Una vez inicializado el control plane es necesario unir cada worker a su correspondiente controlplane usando la información generada por kubeadm init una vez completado el proceso de inicialización del cluster. 
Se asume a partir de aqui que los tres clusters esta operativos y que estamos operando los tres clusters desde una maquina de salto que dispone de un kubeconfig con tres contextos que nos permitirán ir saltando entre cada cluster para facilitar la configuración. 

Asumiendo que la maquina de salto tiene la utilidad Helm instalada, el siguiente paso será instalar el CNI Antrea con el siguiente fichero values.yaml. 

```
# -- Container image to use for Antrea components.
image:
  tag: "v1.11.1"
enablePrometheusMetrics: true
featureGates:
   FlowExporter: true
   AntreaProxy: true
   TraceFlow: true
   NodePortLocal: true
   Egress: true
   AntreaPolicy: true
   Multicluster: true
nodePortLocal:
  # -- Enable the NodePortLocal feature.
  enable: true
  # -- Port range used by NodePortLocal when creating Pod port mappings.
  portRange: "61000-62000"
multicluster:
  # -- Enable Antrea Multi-cluster Gateway to support cross-cluster traffic.
  # This feature is supported only with encap mode.
  enable: true
  # -- The Namespace where Antrea Multi-cluster Controller is running.
  # The default is antrea-agent's Namespace.
  namespace: ""
  # -- Enable StretchedNetworkPolicy which allow Antrea-native policies to select peers
  # from other clusters in a ClusterSet.
  # This feature is supported only with encap mode.
  enableStretchedNetworkPolicy: true
  enablePodToPodConnectivity: true
```
Para ello, primero añadimos el repositorio de antrea desde nuestra jumpbox donde tengamos instalado Helm. 
```
helm repo add antrea https://charts.antrea.io
helm repo update
```
Y, seguidamente, procedemos con la instalación del chart de acuerdo a los valores configurados previamente para asegurar que se habilitan todas las funcionalidades requeridas.
```
helm install antrea antrea/antrea -n kube-system -f values.yaml
````

En la topología multicluster, uno de los clusters actua como LEADER y es el encargado de consolidar los recursos y servicios exportados por cada uno de los MEMBERS. El que actua como LEADER, a su vez, puede participar como MEMBERS. Usando el contexto (kubeconfig) adecuado accedemos al cluster denominado TANZUHISPANO-DB que acutará como LEADER en nuestra topología. Seguidamente procedemos a desplegar los CRDs necesarios (clusterclaims, clustersets, memberclustersannounces, resourceexports y resourceimports) usando el siguiente manifiesto: 

```
kubectl config use-context cl-tanzuhispano-db
kubectl apply -f https://raw.githubusercontent.com/antrea-io/antrea/main/multicluster/build/yamls/antrea-multicluster-leader-global.yml
´´´

Seguidamente instalaremos el controller asi como otros objetos de configuracion necesarios bajo el namespace antrea-multicluster que ha de ser previamente creado

```
kubectl create ns antrea-multicluster
kubectl apply -f https://raw.githubusercontent.com/antrea-io/antrea/main/multicluster/build/yamls/antrea-multicluster-leader-namespaced.yml
```

Una vez hecho esto, es momento de instalar los componentes de multicluster en los clusters MEMBERS. Puesto que queremos que el LEADER actue tambien como MEMBER (es decir, tendrá la capacidad de consumir servicios exportados por otros MEMBERS), tenemos que proceder con la instlación en los tres clusters que conformarán nuestro CLUSTERSET. Todos estos componentes se instalarán en el namespace kube-system por lo que no es necesario crear un ns ad-hoc excepto para el caso del LEADER. En mi caso he usado una maquina de salto que dispone de un fichero de configuración kubeconfig con tres contextos de modo que puedo ir cambiando de contexto desde la misma plataforma. 
Empezamos por instalar los componentes de MEMBER en el cluster TANZUHISPANO-DB que tambien actua como LEADER. 

```
kubectl config use-context cl-tanzuhispano-db
kubectl apply -f https://raw.githubusercontent.com/antrea-io/antrea/main/multicluster/build/yamls/antrea-multicluster-member.yml
´´´
Nos movemos al cluster TANZUHISPANO-01 cambiando de contexto

```
kubectl config use-context cl-tanzuhispano-01
kubectl apply -f https://raw.githubusercontent.com/antrea-io/antrea/main/multicluster/build/yamls/antrea-multicluster-member.yml
´´´

y por último nos movemos al cluster TANZUHISPANO-02 

```
kubectl config use-context cl-tanzuhispano-02
kubectl apply -f https://raw.githubusercontent.com/antrea-io/antrea/main/multicluster/build/yamls/antrea-multicluster-member.yml
´´´

Seguidamente es necesario crear algunos permisos para autorizar la solicitud de membresía de los distintos clusters. Esto lo haremos en el LEADER
```
kubectl config use-context cl-tanzuhispano-db
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: member-tanzuhispano-01
  namespace: antrea-multicluster
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: member-tanzuhispano-02
  namespace: antrea-multicluster
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: member-tanzuhispano-db
  namespace: antrea-multicluster
---
apiVersion: v1
kind: Secret
metadata:
  name: member-tanzuhispano-01-token
  namespace: antrea-multicluster
  annotations:
    kubernetes.io/service-account.name: member-tanzuhispano-01
type: kubernetes.io/service-account-token
---
apiVersion: v1
kind: Secret
metadata:
  name: member-tanzuhispano-02-token
  namespace: antrea-multicluster
  annotations:
    kubernetes.io/service-account.name: member-tanzuhispano-02
type: kubernetes.io/service-account-token
---
apiVersion: v1
kind: Secret
metadata:
  name: member-tanzuhispano-db-token
  namespace: antrea-multicluster
  annotations:
    kubernetes.io/service-account.name: member-tanzuhispano-db
type: kubernetes.io/service-account-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: member-tanzuhispano-01
  namespace: antrea-multicluster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: antrea-mc-member-cluster-role
subjects:
  - kind: ServiceAccount
    name: member-tanzuhispano-01
    namespace: antrea-multicluster
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: member-tanzuhispano-02
  namespace: antrea-multicluster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: antrea-mc-member-cluster-role
subjects:
  - kind: ServiceAccount
    name: member-tanzuhispano-02
    namespace: antrea-multicluster
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: member-tanzuhispano-db
  namespace: antrea-multicluster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: antrea-mc-member-cluster-role
subjects:
  - kind: ServiceAccount
    name: member-tanzuhispano-db
    namespace: antrea-multicluster
EOF
´´´

Seguidamente generamos desde el LEADER los secretos que contendran el token necesario a partir de los secrets previamente creados. 

```
kubectl get secret member-tanzuhispano-01-token -n antrea-multicluster -o yaml | grep -w -e '^apiVersion' -e '^data' -e '^metadata' -e '^ *name:'  -e   '^kind' -e '  ca.crt' -e '  token:' -e '^type' -e '  namespace' | sed -e 's/kubernetes.io\/service-account-token/Opaque/g' -e 's/antrea-multicluster/kube-system/g' >  tanzu-hispano-01-token.yml
kubectl get secret member-tanzuhispano-02-token -n antrea-multicluster -o yaml | grep -w -e '^apiVersion' -e '^data' -e '^metadata' -e '^ *name:'  -e   '^kind' -e '  ca.crt' -e '  token:' -e '^type' -e '  namespace' | sed -e 's/kubernetes.io\/service-account-token/Opaque/g' -e 's/antrea-multicluster/kube-system/g' >  tanzu-hispano-02-token.yml
kubectl get secret member-tanzuhispano-db-token -n antrea-multicluster -o yaml | grep -w -e '^apiVersion' -e '^data' -e '^metadata' -e '^ *name:'  -e   '^kind' -e '  ca.crt' -e '  token:' -e '^type' -e '  namespace' | sed -e 's/kubernetes.io\/service-account-token/Opaque/g' -e 's/antrea-multicluster/kube-system/g' >  tanzu-hispano-db-token.yml
```

Esto gererará un objeto tipo secret con la informacion necesaria para la autenticación de cada MEMBER. El aspecto del fichero generado será similar a el siguiente

```
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJT.... <omitido>
  token: ZXlKaGJHY2lPaUpTVXp.... <omitido>
kind: Secret
metadata:
  name: member-tanzuhispano-01-token
  namespace: kube-system
type: Opaque
´´´
 
 A continutación aplicamos la configuración correspondientemente en cada uno de los miembros del cluster.
 
```
kubectl config use-context cl-tanzuhispano-01
kubectl apply -f tanzu-hispano-01-token.yml 

kubectl config use-context cl-tanzuhispano-02
kubectl apply -f tanzu-hispano-02-token.yml 
 
kubectl config use-context cl-tanzuhispano-db
kubectl apply -f tanzu-hispano-db-token.yml
```

El siguiente paso sería crear el clusterset desde el LEADER. Para ello aplicamos la siguiente configuración

```
kubectl apply -f - <<EOF
apiVersion: multicluster.crd.antrea.io/v1alpha2
kind: ClusterClaim
metadata:
  name: id.k8s.io
  namespace: antrea-multicluster
value: tanzuhispano-cluster-db
---
apiVersion: multicluster.crd.antrea.io/v1alpha2
kind: ClusterClaim
metadata:
  name: clusterset.k8s.io
  namespace: antrea-multicluster
value: tanzuhispano-clusterset
---
apiVersion: multicluster.crd.antrea.io/v1alpha1
kind: ClusterSet
metadata:
  name: tanzuhispano-clusterset
  namespace: antrea-multicluster
spec:
  leaders:
    - clusterID: tanzuhispano-cluster-db
EOF
```

Y en cada uno de los miembros creamos el objeto ClusterClaim para registrarnos en el LEADER. La dirección IP del cluster set corresponderá con la IP del ControlPlane del cluster LEADER. En cada paso cambiamos de cluster para ir aplicando la configuracion correspondiente en cada caso. 

```
kubectl config use-context cl-tanzuhispano-01

kubectl apply -f - <<EOF
apiVersion: multicluster.crd.antrea.io/v1alpha2
kind: ClusterClaim
metadata:
  name: id.k8s.io
  namespace: kube-system
value: tanzuhispano-cluster-01
---
apiVersion: multicluster.crd.antrea.io/v1alpha2
kind: ClusterClaim
metadata:
  name: clusterset.k8s.io
  namespace: kube-system
value: tanzuhispano-clusterset
---
apiVersion: multicluster.crd.antrea.io/v1alpha1
kind: ClusterSet
metadata:
  name: tanzuhispano-clusterset
  namespace: kube-system
spec:
  leaders:
    - clusterID: tanzuhispano-cluster-db
      secret: "member-tanzuhispano-01-token"
      server: "https://10.118.5.200:6443"
  namespace: antrea-multicluster
EOF
´´´
Repetimos para el cluster MEMBER cl-tanzuhispano-02
``` 
kubectl config use-context cl-tanzuhispano-02

kubectl apply -f - <<EOF
apiVersion: multicluster.crd.antrea.io/v1alpha2
kind: ClusterClaim
metadata:
  name: id.k8s.io
  namespace: kube-system
value: tanzuhispano-cluster-02
---
apiVersion: multicluster.crd.antrea.io/v1alpha2
kind: ClusterClaim
metadata:
  name: clusterset.k8s.io
  namespace: kube-system
value: tanzuhispano-clusterset
---
apiVersion: multicluster.crd.antrea.io/v1alpha1
kind: ClusterSet
metadata:
  name: tanzuhispano-clusterset
  namespace: kube-system
spec:
  leaders:
    - clusterID: tanzuhispano-cluster-db
      secret: "member-tanzuhispano-02-token"
      server: "https://10.118.5.200:6443"
  namespace: antrea-multicluster
EOF
´´´
Y, por último para el cluster TANZUHISPANO-DB que tambien actuará como MEMBER además de como LEADER
```
kubectl config use-context cl-tanzuhispano-db
## TANZUHISPANO-DB
kubectl apply -f - <<EOF
apiVersion: multicluster.crd.antrea.io/v1alpha2
kind: ClusterClaim
metadata:
  name: id.k8s.io
  namespace: kube-system
value: tanzuhispano-cluster-db
---
apiVersion: multicluster.crd.antrea.io/v1alpha2
kind: ClusterClaim
metadata:
  name: clusterset.k8s.io
  namespace: kube-system
value: tanzuhispano-clusterset
---
apiVersion: multicluster.crd.antrea.io/v1alpha1
kind: ClusterSet
metadata:
  name: tanzuhispano-clusterset
  namespace: kube-system
spec:
  leaders:
    - clusterID: tanzuhispano-cluster-db
      secret: "member-tanzuhispano-db-token"
      server: "https://10.118.5.200:6443"
  namespace: antrea-multicluster
EOF
```
Si todo fue bien, podemos verificar desde el cluster LEADER usando antctl (es necesario instalar previamente) el estado del CLUSTERSET.

```
$ antctl mc get clusterset tanzuhispano-clusterset -n antrea-multicluster
CLUSTER-ID              NAMESPACE           CLUSTERSET-ID           TYPE  STATUS REASON   
tanzuhispano-cluster-01 antrea-multicluster tanzuhispano-clusterset Ready True   Connected
tanzuhispano-cluster-02 antrea-multicluster tanzuhispano-clusterset Ready True   Connected
tanzuhispano-cluster-db antrea-multicluster tanzuhispano-clusterset Ready True   Connected
```

Con esto ya habríamos creado nuestro entorno multicluster con tres MEMBERS. En mi caso he optado por la configuración en la que se permite la conectividad Pod-To-Pod, la configuración por defecto trabaja con ClusterIP (los cluster importan la información de la IP asignada al servicio en lugar de las IPs de los endpoints/pods). Cualquiera de las dos sería viable en este caso pero yo he optado por habilitar la configuración Pod-To-Pod. Para ello es necesario editar la configuración del multicluster que se guarda en un configmap llamado antrea-mc-controller-config dentro del namespace kube-system. 

```
kubectl config use-context cl-tanzuhispano-01
kubectl edit configmaps -n kube-system antrea-mc-controller-config
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  controller_manager_config.yaml: |
    apiVersion: multicluster.crd.antrea.io/v1alpha1
    kind: MultiClusterConfig
    health:
      healthProbeBindAddress: :8080
    metrics:
      bindAddress: "0"
    webhook:
      port: 9443
    leaderElection:
      leaderElect: false
    serviceCIDR: ""
    podCIDRs:
      - "10.118.96.0/22". # Rango correspondiente de PODCIDR según cada cluster!!! 
    gatewayIPPrecedence: "private"
    endpointIPType: "PodIP"
    enableStretchedNetworkPolicy: true
kind: ConfigMap
```
A continuación reiniciamos el deployment del antrea multicluster controller 
```
kubectl rollout restart deployment -n kube-system antrea-mc-controller
```
Repetir el mismo proceso cambiando de contexto y ajustando los valores del podCIDR según corresponda sin olvidar reiniciar cada vez para que se tomen los nuevos valores.
 
El último paso sería definir qué nodo dentro del cluster actuará como GATEWAY, es decir, el OVS se programará para que el tráfico destinado a un cluster externo egrese e ingrese por uno de los nodos que cumplirá esta tarea especial de encapsular/desencapsular el tráfico cuyo destino se encuentra en un cluster remoto. Esto se consigue a través de una simple anotación en el nodo deseado, en mi caso he tomado el primero de los nodos de cada cluster. Ajustar según la nomemclatura de cada cluster. 







