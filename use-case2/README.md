# Geolocation enforced Cluster responses

In case of high latency sensitive applications, Geolocation based loadbalancing is a typically topology decision maker. To ensure traffic coming in on the closest possible node to the requester, you can use OpenShift (OCP) Ingress sharding and topology labels for Ingress and Gateways to ensure, all services are located in the same topology zone and or subzone to avoid cross region traffic.

## reproducer steps

* for Ingress and HA we need at least two nodes per Zone

* label node1 and node3 by running following command
    ```
    oc label node node1 node3 \
       topology.kubernetes.io/region=datacenter1 \
       topology.kubernetes.io/zone=zone1
    ```
* label node2 and node4 by running following command
    ```
    oc label node node2 node4 \
       topology.kubernetes.io/region=datacenter2 \
       topology.kubernetes.io/region=zone2
    ```

* create a Certificate

Using the wildcard certificate of the base domain from the Cluster is not recommended to avoid known routing errors due to SSL caching in Browsers.
Instead use `cert-manager` and or individual certificates per hostname by running following command:

** create a TLS secret for ingress controller zone1 by running the following command
        ```
        oc -n openshift-ingress create secret tls zone1-tls-secret --cert=... --key=...
        ```

** create a TLS secret for ingress controller zone2 by running the following command
        ```
        oc -n openshift-ingress create secret tls zone2-tls-secret --cert=... --key=...
        ```

* update the default ingress controller to ignore zone1 and zone2 routes by running the following command

```
oc -n openshift-ingress-operator patch ingresscontroller default --type=merge \
 -p '{"spec":{"routeSelector":{"matchExpressions":[{"key":"zonename","operator":"NotIn","values":["zone1","zone2"]}]}}}'
```

* create two new ingress controller to simulate geolocation enforced ingress by running following command

```
oc create -f deploy/ingresscontroller.yml
```

* verify the ingress controllers are ready to be used by running following command

```
oc -n openshift-ingress-operator get IngressController -o yaml | yq -r '.items[]|{"name":.metadata.name,"available":.status.availableReplicas}'
``` 

expected output 

```
name: default
available: 2
name: zone1
available: 2
name: zone2
available: 2
```

* deployment of application(s)
``` 
oc apply -k deploy
``` 

*NOTE* the deployment creates individual services for the routes since we do not have a Load Balancing instance in front of our OCP cluster.
It is not mandatory to create individual services for the routes if you do have a Load Balancer in front as seen in the `simulate one Geo location being down` scenario.

### deployment verification

Use-case1 deploy's 2 instances of the /httpbin application and an individual Gateway for each of them.

```
oc get pods -o wide 
``` 

The output should be similar to the following aligning to your node labels for region and zone

```
NAME                          READY   STATUS    RESTARTS   AGE     IP             NODE       NOMINATED NODE   READINESS GATES
gateway-v1-7f75fc79c8-mbhx2   1/1     Running   0          7m35s   10.133.1.104   node1      <none>           <none>
gateway-v2-6644469697-6wgfj   1/1     Running   0          7m35s   10.132.4.240   node2      <none>           <none>
/httpbin-v1-664f69d9d6-s29w5   2/2     Running   0          7m35s   10.133.1.105   node1      <none>           <none>
/httpbin-v2-5f649dcf9c-gh6wp   2/2     Running   0          7m35s   10.132.4.241   node2      <none>           <none>
```

for a real-life geo-based example you would need another entity (like another Loadbalancer) to be able to have a unique entry point that is
capable to distributed your requests to the Geographically nearest based ingress and zone accordingly.
In the POC we are going to use the NodePort as this separation selector and imagine that some Load Balancer did the magic for us.

* identify the Zone1 nodePort to connect to 

```
oc -n openshift-ingress get service -l router=router-nodeport-zone1 -o yaml | yq -r '.items[0].spec.ports[]|select(.name=="https")|.nodePort'
```

expect an output similar to
```
31426
```

* identify one of the nodes ingress IP address to connect to

```
oc get node -l topology.kubernetes.io/zone=zone1 -o yaml | yq -r '.items[]|.status.addresses[]|select(.type=="InternalIP")|.address'
``` 

expect an output similar to

```
192.168.192.120
192.168.192.121
```

* connect to the specific node and port emulating that your request was drafted through a Geo based load balancing decision

```
export IP=192.168.192.120
export PORT=31426

for x in $(seq 1 10); do \
curl --resolve /httpbin.apps.example.com:${PORT}:${IP} -sk https:///httpbin.apps.example.com:${PORT} | \
  jq -rc '{"name":.env.HOSTNAME,"zone":.env.TOPOLOGY_ZONE}'
done
``` 

expected output 
```
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
```

* identify the Zone2 nodePort to connect to

```
oc -n openshift-ingress get service -l router=router-nodeport-zone2 -o yaml | yq -r '.items[0].spec.ports[]|select(.name=="https")|.nodePort'
```

expect an output similar to (*NOTE* it should be different than the nodePort from zone1)
```
31095
```

* identify one of the nodes ingress IP address to connect to

```
oc get node -l topology.kubernetes.io/zone=zone2 -o yaml | yq -r '.items[]|.status.addresses[]|select(.type=="InternalIP")|.address'
```

expect an output similar to (*NOTE* ip's should be different than the ones from zone1)

```
192.168.192.207
192.168.192.206
```

* connect to the specific node and port emulating that your request was drafted through a Geo based load balancing decision

```
export IP=192.168.192.207
export PORT=31095

for x in $(seq 1 10); do \
curl --resolve /httpbin.apps.example.com:${PORT}:${IP} -sk https:///httpbin.apps.example.com:${PORT} | \
  jq -rc '{"name":.env.HOSTNAME,"zone":.env.TOPOLOGY_ZONE}'
done
``` 

* expected output 

```
{"name":"/httpbin-v2-5f649dcf9c-gh6wp","zone":"zone2"}
{"name":"/httpbin-v2-5f649dcf9c-gh6wp","zone":"zone2"}
{"name":"/httpbin-v2-5f649dcf9c-gh6wp","zone":"zone2"}
{"name":"/httpbin-v2-5f649dcf9c-gh6wp","zone":"zone2"}
{"name":"/httpbin-v2-5f649dcf9c-gh6wp","zone":"zone2"}
{"name":"/httpbin-v2-5f649dcf9c-gh6wp","zone":"zone2"}
{"name":"/httpbin-v2-5f649dcf9c-gh6wp","zone":"zone2"}
{"name":"/httpbin-v2-5f649dcf9c-gh6wp","zone":"zone2"}
{"name":"/httpbin-v2-5f649dcf9c-gh6wp","zone":"zone2"}
{"name":"/httpbin-v2-5f649dcf9c-gh6wp","zone":"zone2"}
```

## simulate one Geo location being down

* *NOTE* these steps are only necessary as we do not have a Load Balancer in front which can distinguish between both ingress controllers.

* shutdown the gateway in zone2 

```
oc scale --replicas=0 deploy/gateway-v2
```

* patch the routes to use the `shared` service instead 

```
oc patch route /httpbin-zone1 --type=merge -p '{"spec":{"to":{"name":"gateway-all"}}}'
oc patch route /httpbin-zone2 --type=merge -p '{"spec":{"to":{"name":"gateway-all"}}}'
```

* connect to the ingress with the settings unchanged meaning hitting zone2 configuration

```
for x in $(seq 1 10); do \
curl --resolve /httpbin.apps.example.com:${PORT}:${IP} -sk https:///httpbin.apps.example.com:${PORT} | \
  jq -rc '{"name":.env.HOSTNAME,"zone":.env.TOPOLOGY_ZONE}'; done
```

expected output 
```
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
{"name":"/httpbin-v1-664f69d9d6-s29w5","zone":"zone1"}
```

## cleanup use-case

```
oc delete -k deploy/
oc delete -f deploy/ingresscontroller.yml
```
