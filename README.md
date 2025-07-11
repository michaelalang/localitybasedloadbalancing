# OpenShift Service Mesh Locality based loadbalancing

With OpenShift Service Mesh 3 (OSSM) multi-primary scenarios are in General Available(GA).

A multi-primary setup means having multiple kubernetes clusters, each with its own Istio control plane (Istiod), allowing for independent management and fault isolation. This configuration is ideal when you need to manage services across different regions, cloud providers, or have varying security requirements

Locality based loadbalancing can be used even without a multi-primary deployment.
The basic requiremets are:

* kubernetes topology labels (topology.kubernetes.io/region, topology.kubernetes.io/zone)
* a DestinationRule CustomResource(CR) for the endpoints with the configuration `localityLbSetting`.
* Gateway injection for the Workload to be reached.
* *Optional* Router sharding for DataCentre(DC) specific ingress availability (Geolocation based LoadBalancing)
* *Optional* Region or DC aware LoadBalancing Frontend for early locality based loadbalancing

**Note** 
kubernetes term for the use-cases is locality based **routing** where ServiceMesh and in particular Envoy terminology is locality based **loadbalancing**

## Labeling your Cluster nodes for locality based loadbalancing

**NOTE** kuberentes topology labels will have an impact of your workloads when being deployed and shall only be set in alignment with the infrastructure Team and any external infrastructure provider like VSphere to comply with topology configurations through all management tools.

Following labels can be applied to make your workload and your ServiceMesh loadbalancing locality aware:
* topology.kubernetes.io/region
* topology.kubernetes.io/zone
* topology.istio.io/subzone

Envoy uses topology labels automatically for ServiceMesh locality based loadbalancing configurations from the underlying nodes where the pods will be scheduled to. 

Label your nodes by running following commands
```
oc label nodes node1 node2 node3 \
  topology.kubernetes.io/region=region1 \
  topology.kubernetes.io/zone=zone1 \
  topology.istio.io/subzone=subzone1 
```
```
oc label nodes node4 node5 node6 \
  topology.kubernetes.io/region=region1 \
  topology.kubernetes.io/zone=zone1 \
  topology.istio.io/subzone=subzone2 
```  

## Deployment nodeSelector for topology aware routing

In the deployment CR you can specify a nodeSelector (simpliest form) to schedule workloads to a specific region/zone/subzone. The example will use only nodeSelector with labels and for more complex classifications review [Topology aware routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/) and [Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/) documentation.

Example Deployment snippet Deployment 1:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin1
...
spec:
  template:
    metadata:
      labels:
        app: httpbin
        version: v1
        sidecar.istio.io/inject: true
    spec:
      nodeSelector:
        topology.istio.io/subzone: subzone1                     
        topology.kubernetes.io/zone: zone1
        topology.kubernetes.io/region: region1
     ...
```

Example Deployment snippet Deployment 2:
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin2
...
spec:
  template:
    metadata:
      labels:
        app: httpbin
        version: v2
        sidecar.istio.io/inject: true
    spec:
      nodeSelector:
        topology.istio.io/subzone: subzone2
        topology.kubernetes.io/zone: zone1
        topology.kubernetes.io/region: region1
     ...
```

The `service` selector will still match both of the instances as the decission is for ServiceMesh to prefer region/zone/subzone1 or region/zone/subzone2

Example Service snippet for both Deployments:
```
apiVersion: v1
kind: Service
...
spec:
  selector:
    app: httpbin
...
```

## Configuring the DestinationRule for Locality based loadbalancing

A DestinationRule defines traffic policies that are applied to traffic destined for a service after it has been routed by a VirtualService.
Configure a DestinationRule for locality based loadbalancing by extending the configuration with the `trafficPolicy` object and furthermore add the desired `localityLBSetting`

Example DestinationRule snippet:

```
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: httpbin
spec:
  host: httpbin.namespace.svc.cluster.local
  subsets: #(1)
  - labels:
      version: v1
    name: v1
  - labels:
      version: v2
    name: v2
  trafficPolicy:
    connectionPool: 
      http:
        maxRequestsPerConnection: 1 #(2)
    loadBalancer:
      consistentHash:
        useSourceIp: true #(3)
      localityLbSetting:
        enabled: true
        failover: #(4)
        - from: region1/zone1/subzone1
          to: region1/zone1/subzone2
    outlierDetection: #(5)
      baseEjectionTime: 1s
      consecutive5xxErrors: 0
      consecutiveGatewayErrors: 0
      interval: 100ms
      maxEjectionPercent: 0
```

1) DestinationRule subsets are to classify the same workload name into different version. In this scenario the Application versions are equal and serve the purpose for locality classification.
2) maxRequestsPerConnection is mandatory to ensure locality based loadbalancing will switch over and not re-use the existing connection for subsequent requests.
3) Without sticking to the source IP of the ServiceMesh Gateway and in particular in multi-master scenarios, the consistency cannot be guaranteed without this setting to true.
4) The failover scenarios you want to follow see [Gateway alignment](#gateway-alignment)
5) OutlierDetection ensure's Envoy proxies will not retry an endpoint in a failed or empty zone.

### Gateway alignment

Envoys [documentation](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/locality_weight.html) outlines, locality based loadbalancing will try to deiliver within it's own zone until there are no healthy endpoints available. 
In scenarios were your Gateway forwards the traffic to region/zone/subzone2 instead of region/zone/subzone1 means that your Gateway is not part of the region/zone/subzone1 eventhough all endpoints are up and healthy.

Adjust your Gateway injection template accordingly to your needs and preferred region/zone/subzone setup.

Example Gateway Deployment snippet: 

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gateway
spec:
  selector:
    matchLabels:
      istio: gateway
  template:
    metadata:
      annotations:
        inject.istio.io/templates: gateway
      labels:
        istio: gateway
        sidecar.istio.io/inject: "true"
    spec:
      nodeSelector:
        topology.istio.io/subzone: subzone1
        topology.kubernetes.io/zone: zone1
        topology.kubernetes.io/region: region1
      containers:
      - name: istio-proxy
        image: auto
```

Verify the Endpoint configured topology by running the following command:

```
$ istioctl  pc all $(oc get pods -l istio=gateway -o name)  | grep 'endpoint/' | grep '|8080||httpbin'
endpoint/10.132.2.152:8080                                HEALTHY     region1/zone1/subzone1         outbound|8080||httpbin.namespace.svc.cluster.local
endpoint/10.133.0.106:8080                                HEALTHY     region1/zone1/subzone2         outbound|8080||httpbin.namespace.svc.cluster.local 
```

# use cases
## Data Center or Cluster Fire section requirements
 
A common toplogy spread you might face in your Data Center is Cluster alignement to fire sections. The Kubernetes labels for topology should be sufficient and if they do not align with your ServiceMesh topology rollout, you can utilize the topology.istio.io/subzone label in addition.

[POC](use-case1/README.md)

## Geolocation enforced Cluster responses

In case of high latency sensitive applications, Geolocation based loadbalancing is a typically topology decision maker. To ensure traffic coming in on the closest possible node to the requester, you can use OpenShift (OCP) Ingress sharding and topology labels for Ingress and Gateways to ensure, all services are located in the same topology zone and or subzone to avoid cross region traffic.

Setting up a Demo with two OpenShift Router shards and two Istio Gateways show's how geolocation enforced routing will stick to the closes possible resource.
(in the Demo, the header `zone` will simulate Geo IP differences)

[POC](use-case2/README.md)

![Show casing geo based locality loadbalancing](localitybasedloadbalancing-geo-split.gif)


## Low Latency enforced Application responses by grouping Application and Backends into topology zones

To ensure best possible performance between Applications and or their backends, topolgy spread classification into the same region/zone/subzone will guarantee that the closest possible healthy endpoint will be used to serve a request.

## Multi-primary ServiceMesh cluster scenarios

Extending the already mentioned use cases to a mulit-primary Service Mesh setup, we can use topology labels to ensure the chain of responsible services within one Cluster, followed by other Clusters in the multi-primary scenario to increase resilience and fault tolerance.

Following snipped shows a multi-primary failover configuration:

```
  trafficPolicy:
    connectionPool: 
      http:
        maxRequestsPerConnection: 1 (2)
    loadBalancer:
      consistentHash:
        useSourceIp: true (3)
      localityLbSetting:
        enabled: true
        failover: (4)
        - from: cluster1/zone1/subzone1
          to: cluster1/zone1/subzone2
        - from: cluster2/zone1/subzone1
          to: cluster2/zone1/subzone2
        - from: cluster2/zone1/subzone2
          to: cluster3/zone1/subzone1
...
```

*NOTE* According to the [matrix](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/locality_weight#locality-weighted-load-balancing) of envoys `locality_weight` and as shown below, you will see different responses according to the endpoints available in a locality zone.

With multi-primary locality load balancing ensure to have following configuration snippet in your DestinationRule to mitigate the issue:

```
trafficPolicy:
  loadBalancer:
    consistentHash:
      useSourceIp: true
``` 

Alternatively scaling the east-west gateways accordingly to satistfy the healthy endpoint percentage as documented in envoys [matrix](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/locality_weight#locality-weighted-load-balancing).

[POC](use-case3/README.md)

![Show casing locality based multi-cluster loadbalancing](localitybasedloadbalancing-cluster-failover.gif)

## Multi-primary ServiceMesh cluster scenario with priority failover between Cluster

With a special requirement to failover `zone1` from cluster *one* to `zone1` on cluster *two* and worst case back to cluster *one* `zone2` we cannot succeed with locality based loabalancing in `failover` as the subzone endpoints in zone2 are still healthy if zone1 endpoints are all down. 

`failoverPriority` can be utilized to get the scenario working as expected. Classifying the priorities to exclude the zone and to match the version instead:

With following priorty classification for the workload 

* region/zone1/subzone - cluster1/version=v1
* region/zone1/subzone - cluster2/version=v1
* region/zone2/subzone - cluster1/version=v2

Defining that *Cluster1* will serve requests from application with label `version=v1` until no endpoints are available. 
Failing over to *Cluster2* serving requests from application with label `version=v1` until no endpoints are available.
Failing over to *Cluster1* serving requests from application with label `version=v2` until no endpoints are available.

the DestinationRule configuration snippet for this scenario is:

```
      localityLbSetting:
        enabled: true
        failoverPriority:
        - topology.istio.io/subzone
        - version=v1
```

There is no need to explicitly list more than the `version=v1` priority as the Zone endpoints for Cluster1 and Cluster2 will be utilized comprehensive due to the `toplogy.istio.io/subzone` label classification.


[POC](use-case4/README.md)

![Show casing locality based failoverPriority](localitybasedloadbalancing-prioriy-based.gif)

# Live Demo locality based loadbalancing

![Show casing locality based loadbalancing](localitybasedloadbalancing.gif)
