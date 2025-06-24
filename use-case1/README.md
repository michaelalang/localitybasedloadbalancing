# Data Center or Cluster Fire section requirements

A common toplogy spread you might face in your Data Center is Cluster alignement to fire sections. The Kubernetes labels for topology should be sufficient and if they do not align with your ServiceMesh topology rollout, you can utilize the topology.istio.io/subzone label in addition.

## reproducer steps

* label node1 by running following command
    ```
    oc label node node1 \
       topology.kubernetes.io/region=datacenter1 \
       topology.kubernetes.io/zone=firezone1 
    ```
* label node2 by running following command
    oc label node node2 \
       topology.kubernetes.io/region=datacenter1 \
       topology.kubernetes.io/region=firezone2
    ``` 

* create a Certificate 

Using the wildcard certificate of the base domain from the Cluster is not recommended to avoid known routing errors due to SSL caching in Browsers.
Instead use `cert-manager` and or individual certificates per hostname by running following command:

```
oc create secret tls gateway-v1-secret --cert=... --key=... 
``` 

* Deployment of application(s)
``` 
oc apply -k deploy
``` 

### Deployment verification

The Use-case1 deploy's 4 instances (v1-v4) of the httpbin application.
* The odd numbers (1,3) are in one fire section (datacenter1/firezone1)    
* The even numbers (2,4) are in the other fire section (datacenter1/firezone2)

```
oc get pods -o wide 
``` 

The output should be similar to the following aligning to your node labels for region and zone

```
NAME                           READY   STATUS            RESTARTS   AGE    IP             NODE       NOMINATED NODE   READINESS GATES
/httpbin-v1-69f75cb6df-mwpdq    2/2     Running           0          4s    10.132.3.117   node1   	<none>           <none>
/httpbin-v2-6dd74c947b-g9swr    2/2     Running   	 0          4s    10.133.1.102   node2   	<none>           <none>
/httpbin-v3-64489bbf7-rvfwq     2/2     Running		 0          4s    10.134.3.4     node1   	<none>           <none>
/httpbin-v4-75b5bf94d9-k9dhw    2/2     Running           0          4s    10.134.0.95    node2       	<none>           <none>
gateway-v1-67b8b7cc76-62clr    1/1     Running           0          4s    10.132.3.141   node1   	<none>           <none>
gateway-v2-8f9dd6ff7-mtdn2     1/1     Running           0          4s    10.132.2.227   node2   	<none>           <none>
```

Try connecting to the backend from the instances v1 and v2 with following command

```
oc exec -ti deploy//httpbin-v1 -c /httpbin -- \
 python3 -c 'import requests; env = requests.get("http://backend:8080").json()["env"] ; print(f"""{env["HOSTNAME"]} {env["TOPOLOGY_ZONE"]}""")'
```
expected output
```
/httpbin-v3-64489bbf7-rvfwq firezone1
``` 

```
oc exec -ti deploy//httpbin-v2 -c /httpbin -- \
 python3 -c 'import requests; env = requests.get("http://backend:8080").json()["env"] ; print(f"""{env["HOSTNAME"]} {env["TOPOLOGY_ZONE"]}""")'
```
expected output
```
/httpbin-v4-75b5bf94d9-k9dhw firezone2
```

and of course as well from an external client the same is expected
```
curl -sk -X POST -H 'Content-Type: application/json' \
 -d '[{"url":["http://backend:8080"],"method":"GET","headers":{}}]' \
 https:///httpbin.apps.example.com/tracefwd | \
 jq -rc '{"name":.response.env.HOSTNAME,"zone":.response.env.TOPOLOGY_ZONE}'
```

expected output 
```
{"name":"/httpbin-v3-64489bbf7-rvfwq","zone":"firezone1"}
```

### Firezone failover simulation

To keep the POC simple we did not utilize sharding Ingress which would provide the capability to use individual ingress Nodes and Pods per Firezone.
Let's set the gateway service instead to match to firezone2 Istio ingress.

```
oc patch --type merge -p '{"spec":{"to":{"name": "gateway-2"}}}' route /httpbin
``` 

now verify that the Firezone to is responding and staying in the zone 
```
curl -sk -X POST -H 'Content-Type: application/json' \
 -d '[{"url":["http://backend:8080"],"method":"GET","headers":{}}]' \
 https:///httpbin.apps.example.com/tracefwd | \
 jq -rc '{"name":.response.env.HOSTNAME,"zone":.response.env.TOPOLOGY_ZONE}'
```

expected output
```
{"name":"/httpbin-v4-75b5bf94d9-k9dhw","zone":"firezone2"}
```


## cleanup use-case 

```
oc delete -k deploy/
``` 
