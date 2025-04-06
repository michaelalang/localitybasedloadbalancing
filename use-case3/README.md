# Multi-master ServiceMesh cluster scenarios

Extending the already mentioned use cases to a mulit-master Service Mesh setup, we can use topology labels to ensure the chain of responsible services within one Cluster, followed by other Clusters in the multi-master scenario.


## reproducer steps

* required upfront two cluster in multi-master configuration 

* label nodes
    ```
    # cluster 1, two nodes with each one in a zone
    oc --context cluster1 label node node1 \
       topology.kubernetes.io/region=datacenter1 \
       toptopology.kubernetes.io/zoneology.kubernetes.io/zone=zone1
    oc --context cluster1 label node node2 \
       topology.kubernetes.io/region=datacenter1 \
       topology.kubernetes.io/zone=zone2
    # cluster 2, two nodes with each one in a zone
    oc --context cluster2 label node node1 \
       topology.kubernetes.io/region=datacenter2 \
       topology.kubernetes.io/region=zone1
    oc --context cluster2 label node node2 \
       topology.kubernetes.io/region=datacenter2 \
       topology.kubernetes.io/region=zone2
    ```

* deployment of application(s)
``` 
oc apply -k deploy
``` 

### deployment verification

Use-case3 deploy's 2 instances of the mockbin application in each cluster with a related gateway.

```
oc --context cluster1 get pods -o wide 
oc --context cluster2 get pods -o wide
``` 

The output should be similar to the following aligning to your node labels for region and zone

```
# cluster1
NAME                          READY   STATUS    RESTARTS   AGE
gateway-5c8b9899c-q9h9x       1/1     Running   0          16s
mockbin-v1-698f54f95c-t8tq8   2/2     Running   0          16s
mockbin-v2-8ff547f84-ch8c7    2/2     Running   0          16s

# cluster2
NAME                          READY   STATUS    RESTARTS   AGE
gateway-595c74c75d-mdckm      1/1     Running   0          16s
mockbin-v1-77d7c8b575-fwngr   2/2     Running   0          16s
mockbin-v2-848d5dc4fd-2pfxf   2/2     Running   0          16s
```

Executing to each clusters ingress should return one instance pod name of the related cluster accordingly.

```
# find the IP address of the ingress router cluster1
export IP=$(host $(oc --context cluster1 get route mockbin -o yaml | yq -r '.status.ingress[].routerCanonicalHostname')| awk ' { print $NF }')
echo curl --resolve mockbin.apps.example.com:443:${IP} https://mockbin.apps.example.com -s | bash | jq -r .env.HOSTNAME
mockbin-v1-698f54f95c-t8tq8

# find the IP address of the ingress router cluster2
export IP=$(host $(oc --context cluster2 get route mockbin -o yaml | yq -r '.status.ingress[].routerCanonicalHostname')| awk ' { print $NF }')
echo curl --resolve mockbin.apps.example.com:443:${IP} https://mockbin.apps.example.com -s | bash | jq -r .env.HOSTNAME
mockbin-v2-848d5dc4fd-2pfxf
```

## Failover Cluster1 

* open a second terminal and curl the service constantly accessing the gateway of Cluster1 

```
export IP=$(host $(oc --context cluster1 get route mockbin -o yaml | yq -r '.status.ingress[].routerCanonicalHostname')| awk ' { print $NF }')
while true ; do echo curl --resolve mockbin.apps.example.com:443:${IP} https://mockbin.apps.example.com -s | bash | jq -r .env.HOSTNAME ; sleep 1 ; done
mockbin-v1-698f54f95c-t8tq8
mockbin-v1-698f54f95c-t8tq8
...
``` 

* Simulating a full Application outage on Cluster1 means v1 and v2 are both broken so we shut it down

```
oc --context cluster1 scale --replicas=0 deploy/mockbin-v1 deploy/mockbin-v2
```

* check that your curl ping in the other Terminal has failovered to Cluster2 accordingly.

```
mockbin-v1-698f54f95c-t8tq8
mockbin-v1-698f54f95c-t8tq8
mockbin-v1-77d7c8b575-fwngr
mockbin-v1-77d7c8b575-fwngr
mockbin-v1-77d7c8b575-fwngr
```

* scale back the application in Cluster1 and see the curl ping failing back to Cluster1

```
oc --context cluster1 scale --replicas=1 deploy/mockbin-v1 deploy/mockbin-v2
oc --context central -n lb-multi get pods
```

expected output
```
NAME                          READY   STATUS    RESTARTS   AGE
gateway-5c8b9899c-q9h9x       1/1     Running   0          26m
mockbin-v1-698f54f95c-w7snv   2/2     Running   0          47s
mockbin-v2-8ff547f84-78wm9    2/2     Running   0          47s
``` 

expected output of the curl ping
```
mockbin-v1-77d7c8b575-fwngr
mockbin-v1-77d7c8b575-fwngr
mockbin-v1-77d7c8b575-fwngr
mockbin-v1-698f54f95c-w7snv
mockbin-v1-698f54f95c-w7snv
mockbin-v1-698f54f95c-w7snv
mockbin-v1-698f54f95c-w7snv
```

## cleanup use-case

```
oc --context cluster1 delete -k deploy/
oc --context cluster2 delete -k deploy/
```
