# Multi-master ServiceMesh cluster scenarios with priority between Cluster

With a special requirement to failover zone1 from cluster one to zone1 on cluster two and worst case back to cluster one zone2 we cannot succeed with locality based loabalancing in failover as the subzone endpoints in zone2 are still healthy if zone1 endpoints are all down. failoverPriority can be utilized to get the scenario working as expected. Therefor we classify the priorities to exclude the zone and to match the version instead:

* region/zone1/subzone - cluster1/version=v1
* region/zone1/subzone - cluster2/version=v1
* region/zone2/subzone - cluster1/version=v2

## reproducer steps

* required upfront two cluster in multi-master configuration 

* label nodes
    ```
    # cluster 1, two nodes with each one in a zone
    oc --context cluster1 label node node1 \
       topology.kubernetes.io/region=datacenter1 \
       topology.kubernetes.io/zone=zone1 \
       topology.istio.io/subzone=zone1 
    oc --context cluster1 label node node2 \
       topology.kubernetes.io/region=datacenter1 \
       topology.kubernetes.io/zone=zone1 \
       topology.istio.io/subzone=zone2
    # cluster 2, two nodes with each one in a zone
    oc --context cluster2 label node node1 \
       topology.kubernetes.io/region=datacenter1 \
       topology.kubernetes.io/region=zone1
       topology.istio.io/subzone=zone1 
    ```

* deployment of application(s)
``` 
oc --context cluster1 apply -k deploy
oc --context cluster2 apply -k deploy
``` 

### deployment verification

Use-case4 deploy's 2 instances of the mockbin application in each cluster with a related gateway.
Event though the second instance on the second cluster is not expected to come up, it's deployed due to simplicity.


```
oc --context cluster1 get pods -o wide 
oc --context cluster2 get pods -o wide
``` 

The output should be similar to the following aligning to your node labels for region and zone

```
# cluster1
NAME                          READY   STATUS    RESTARTS   AGE     IP             NODE       NOMINATED NODE   READINESS GATES
gateway-749ddb47b9-82lx7      1/1     Running   0          12m     10.133.1.115   node03   <none>           <none>
mockbin-v1-664f69d9d6-njk8l   2/2     Running   0          12m     10.133.1.116   node03   <none>           <none>
mockbin-v2-6d6775f77c-m47bp   2/2     Running   0          4m30s   10.132.5.37    node10   <none>           <none>

# cluster2
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE   NOMINATED NODE   READINESS GATES
gateway-65b9b6cf86-ssxdl      1/1     Running   0          12m   10.136.0.172   node01   <none>           <none>
mockbin-v1-69f745dbd7-qptgq   2/2     Running   0          12m   10.136.0.173   node01   <none>           <none>
```

Executing to each clusters ingress should return one instance pod name of the related cluster accordingly.

```
# find the IP address of the ingress router cluster1
export IP=$(host $(oc --context cluster1 get route mockbin -o yaml | yq -r '.status.ingress[].routerCanonicalHostname')| awk ' { print $NF }')
echo curl --resolve mockbin.apps.example.com:443:${IP} https://mockbin.apps.example.com -s | bash | jq -r .env.HOSTNAME
mockbin-v1-664f69d9d6-njk8l

# find the IP address of the ingress router cluster2
export IP=$(host $(oc --context cluster2 get route mockbin -o yaml | yq -r '.status.ingress[].routerCanonicalHostname')| awk ' { print $NF }')
echo curl --resolve mockbin.apps.example.com:443:${IP} https://mockbin.apps.example.com -s | bash | jq -r .env.HOSTNAME
mockbin-v1-69f745dbd7-qptgq
```

## Failover Cluster1-Zone1 to Cluster2-Zone1 

* open a second terminal and curl the service constantly accessing the gateway of Cluster1 

```
export IP=$(host $(oc --context cluster1 get route mockbin -o yaml | yq -r '.status.ingress[].routerCanonicalHostname')| awk ' { print $NF }')
while true ; do echo curl --resolve mockbin.apps.example.com:443:${IP} https://mockbin.apps.example.com -s | bash | jq -r .env.HOSTNAME ; sleep 1 ; done
...
``` 

* Simulating a full Application outage on Cluster1 means v1 and v2 are both broken so we shut it down

```
oc --context cluster1 scale --replicas=0 deploy/mockbin-v1
```

* check that your curl ping in the other Terminal has failovered to Cluster2 accordingly.

```
mockbin-v1-664f69d9d6-njk8l
mockbin-v1-664f69d9d6-njk8l
mockbin-v1-664f69d9d6-njk8l
mockbin-v1-69f745dbd7-qptgq
mockbin-v1-69f745dbd7-qptgq
mockbin-v1-69f745dbd7-qptgq
```

* furthermore fail the Service on Cluster 2 as well

```
oc --context cluster2 scale --replicas=0 deploy/mockbin-v1
```

expected output of the curl ping
```
mockbin-v1-69f745dbd7-qptgq
mockbin-v1-69f745dbd7-qptgq
mockbin-v1-69f745dbd7-qptgq
mockbin-v2-6d6775f77c-m47bp
mockbin-v2-6d6775f77c-m47bp
mockbin-v2-6d6775f77c-m47bp
```

* after Cluster 1 Zone 2 responses, respawn the Cluster 2 service

```
oc --context cluster2 scale --replicas=1 deploy/mockbin-v1
``` 

* the response should change once the pod is ready
```
oc --context cluster2 get pods 
```

Expected output
```
NAME                          READY   STATUS    RESTARTS   AGE
gateway-65b9b6cf86-ssxdl      1/1     Running   0          18m
mockbin-v1-69f745dbd7-4k9bd   2/2     Running   0          45s
``` 

* the curl ping changes accordingly to the new pod in Cluster 2

```
mockbin-v2-6d6775f77c-m47bp
mockbin-v2-6d6775f77c-m47bp
mockbin-v1-69f745dbd7-4k9bd
mockbin-v1-69f745dbd7-4k9bd
mockbin-v1-69f745dbd7-4k9bd
``` 

* and also scale the Zone1 Cluster 1 Service back up

```
oc --context cluster1 scale --replicas=1 deploy/mockbin-v1
```

* the response should change once the pod is ready
```
oc --context cluster2 get pods
```

Expected output
```
NAME                          READY   STATUS    RESTARTS   AGE
gateway-749ddb47b9-82lx7      1/1     Running   0          20m
mockbin-v1-664f69d9d6-9jjgx   2/2     Running   0          16s
mockbin-v2-6d6775f77c-m47bp   2/2     Running   0          12m
```

* the curl ping changes accordingly to the new pod in Cluster 1 Zone 1

```
mockbin-v1-69f745dbd7-4k9bd
mockbin-v1-69f745dbd7-4k9bd
mockbin-v1-69f745dbd7-4k9bd
mockbin-v1-664f69d9d6-9jjgx
mockbin-v1-664f69d9d6-9jjgx
mockbin-v1-664f69d9d6-9jjgx
```

## cleanup use-case

```
oc --context cluster1 delete -k deploy-c1/
oc --context cluster2 delete -k deploy-c2/
```
