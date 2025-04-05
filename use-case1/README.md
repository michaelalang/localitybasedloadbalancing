# Data Center or Cluster Fire section requirements

A typically toplogy spread will be if your Data Center and Cluster need to be aligned into different fire sections. The Kubernetes labels for topology should be sufficient and if they do not align with your ServiceMesh topology rollout, you can utilize the topology.istio.io/subzone label in addition.


## reproducer steps

* label nodes 
    ```
    oc label node node1 \
       topology.kubernetes.io/region=datacenter1 \
       topology.kubernetes.io/zone=firezone1 
    oc label node node2 \
       topology.kubernetes.io/region=datacenter1 \
       topology.kubernetes.io/region=firezone2
    ``` 

* create a Certificate 

for the purpose of the Demo you can copy the wildcard certificate of the Cluster but I highly recommend to 
instead use `cert-manager` and individual certificates per hostname to avoid known routing errors due to SSL caching.

    ```
    oc create secret tls gateway-v1-secret --cert=... --key=... 
    ``` 

* deployment of application(s)
    ``` 
    oc apply -k deploy
    ``` 

### deployment verification

Use-case1 deploy's 4 instances (v1-v4) of the mockbin application.
* The odd numbers (1,3) are in one fire section (datacenter1/firezone1)    
* The even numbers (2,4) are in the other fire section (datacenter1/firezone2)

```
oc get pods -o wide 
``` 

The output should be similar to the following aligning to your node labels for region and zone

```
NAME                           READY   STATUS            RESTARTS   AGE    IP             NODE       NOMINATED NODE   READINESS GATES
mockbin-v1-69f75cb6df-mwpdq    2/2     Running           0          4s    10.132.3.117   node1   	<none>           <none>
mockbin-v2-6dd74c947b-g9swr    2/2     Running   	 0          4s    10.133.1.102   node2   	<none>           <none>
mockbin-v3-64489bbf7-rvfwq     2/2     Running		 0          4s    10.134.3.4     node1   	<none>           <none>
mockbin-v4-75b5bf94d9-k9dhw    2/2     Running           0          4s    10.134.0.95    node2       	<none>           <none>
gateway-v1-67b8b7cc76-62clr    1/1     Running           0          4s    10.132.3.141   node1   	<none>           <none>
gateway-v2-8f9dd6ff7-mtdn2     1/1     Running           0          4s    10.132.2.227   node2   	<none>           <none>
```

Try connecting to the backend from the instances v1 and v2 with following command

```
oc exec -ti deploy/mockbin-v1 -c mockbin -- \
 python3 -c 'import requests; env = requests.get("http://backend:8080").json()["env"] ; print(f"""{env["HOSTNAME"]} {env["TOPOLOGY_ZONE"]}""")'
```
expected output
```
mockbin-v3-64489bbf7-rvfwq firezone1
``` 

```
oc exec -ti deploy/mockbin-v2 -c mockbin -- \
 python3 -c 'import requests; env = requests.get("http://backend:8080").json()["env"] ; print(f"""{env["HOSTNAME"]} {env["TOPOLOGY_ZONE"]}""")'
```
expected output
```
mockbin-v4-75b5bf94d9-k9dhw firezone2
```

and of course as well from an external client the same is expected
```
curl -sk -X POST -H 'Content-Type: application/json' \
 -d '[{"url":["http://backend:8080"],"method":"GET","headers":{}}]' \
 https://mockbin.apps.example.com/tracefwd | \
 jq -rc '{"name":.response.env.HOSTNAME,"zone":.response.env.TOPOLOGY_ZONE}'
```

expected output 
```
{"name":"mockbin-v3-64489bbf7-rvfwq","zone":"firezone1"}
```

### Firezone failover simulation

To keep the POC simple we did not utilize sharding Ingress which would provide the capability to use individual ingress Nodes and Pods per Firezone.
Let's set the gateway service instead to match to firezone2 Istio ingress.

```
oc patch --type merge -p '{"spec":{"to":{"name": "gateway-2"}}}' route mockbin
``` 

now verify that the Firezone to is responding and staying in the zone 
```
curl -sk -X POST -H 'Content-Type: application/json' \
 -d '[{"url":["http://backend:8080"],"method":"GET","headers":{}}]' \
 https://mockbin.apps.example.com/tracefwd | \
 jq -rc '{"name":.response.env.HOSTNAME,"zone":.response.env.TOPOLOGY_ZONE}'
```

expected output
```
{"name":"mockbin-v4-75b5bf94d9-k9dhw","zone":"firezone2"}
```


## cleanup use-case 

```
oc delete -k deploy/
``` 
