# Tanzuhispano

Se trata de crear una topología que combine la funcionalidad Multicluster de ANTREA con el GeoBalanceo de carga y los servicios de Ingress (AVI). La topología buscada esta represntada en la siguiente figura.

<img width="1088" alt="image" src="https://github.com/jhasensio/tanzuhispano/assets/37454410/3caaa3d5-cdbe-4f73-94bb-c47a0ec95612">


El primer paso sería inicializar los clusters. En este caso he usado clusters vanilla 1.24 pero cualquier versión sería adecuada. Es importante utilizar redes de Pod y de Services NO SOLAPADAS. En este caso una manera de inicializar los clusters para respetar el direccionamiento indicado en el dibujo de arriba sería. 

 CLUSTER TANZUHISPANO-01
 
```
      kubeadm init --service-cidr=10.228.96.0/22 --pod-network-cidr=10.118.96.0/22
```

 CLUSTER TANZUHISPANO-DB
 
```
      kubeadm init --service-cidr=10.228.112.0/22 --pod-network-cidr=10.118.112.0/22
```

 CLUSTER TANZUHISPANO-02
 
```
      kubeadm init --service-cidr=10.228.128.0/22 --pod-network-cidr=10.118.128.0/22
```

Seguidamente instalamos Antrea usando Helm con el siguiente fichero values.yaml 

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
Para ello, primero añadimos el repositorio de antrea
```
helm repo add antrea https://charts.antrea.io
helm repo update
```
Y, seguidamente, procedemos con la instalación del chart de acuerdo a los valores configurados
```
helm install antrea antrea/antrea -n kube-system -f values.yaml
````

En la topología multilcluster, uno de los clusters actua como LEADER y es el encargado de procesar los service-exports de los distintos miembros y 
