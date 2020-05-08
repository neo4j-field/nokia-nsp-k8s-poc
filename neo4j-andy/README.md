# neo4j-causal-cluster-k8s

Deploying single Neo4j Causal Cluster across 2 Kubernetes cluster

Main Purpose: To be able to deploy Neo4j Causal Cluster across 2 Kubernetes Cluster in different regions. This is required to be able to cater to DR.

This approach uses Load Balancers to route connections from the other K8s cluster.

n.b. the provided yaml creates 3 Neo4j cores in (Kubernetes) "cluster 1" and 2 Neo4j cores in (Kubernetes) "cluster 2". All cores are part of a single Neo4j Causal Cluster.

## Prerequisites

### Neo4j 3.5 Enterprise Edition Docker Image

Your kubernetes nodes need to be able to pull a Neo4j 3.5 Enterprise Edition Docker Image. The official image is available from DockerHub at `neo4j:3.5`.

If your nodes cannot pull from DockerHub directly then you should fetch the official docker image and publish it to a private docker repo that your nodes can access.

For example, to copy the official `3.5.17` image to a private Google Cloud Container Repository you can use the following commands:

```
# see https://cloud.google.com/container-registry/docs/pushing-and-pulling
export GCLOUD_PROJECT_NAME="my-project-name"
gcloud auth configure-docker
docker pull neo4j:3.5.17-enterprise
docker tag neo4j:3.5.17-enterprise us.gcr.io/${GCLOUD_PROJECT_NAME}/neo4j:3.5.17
docker push eu.gcr.io/${GCLOUD_PROJECT_NAME}/neo4j:3.5.17
```

## Starting a Neo4j Causal Cluster

### Step 1 - Create the Discovery Load Balancers

Create the load balancers in each cluster. They won't actually have anything to load balance over BUT if we create them they will be assigned IP addresses which we need to use in the next step.

n.b. depending on your infrastructure provider / configuration you may need to modify these to specify a static external IP address.

#### Kubernetes Cluster 1:
```
kubectl apply -f k8s-1-discovery-load-balancers.yaml
```

#### Kubernetes Cluster 2:
```
kubectl apply -f k8s-2-discovery-load-balancers.yaml
```

### Step 2 - Configure Discovery Load Balancer IPs

We need to get the external IP addresses of the load balancers and use them to fill in our yaml files.

We will use kubectl to get the External IP address for the discovery load balancers and then update the yaml with the IP addresses.

n.b. the IP addresses from Cluster 1 go in the confguration (yaml) files for Cluster 2 and vice-versa.

#### K8s Cluster 1:

```
$ kubectl get svc -l neo4j.com/role=external-cluster-members -o custom-columns=NAME:.metadata.name,EXTERNAL-IP:..ingress..ip
NAME                                        EXTERNAL-IP
k8s-1-graph-name-neo4j-discovery-public-0   34.91.174.200
k8s-1-graph-name-neo4j-discovery-public-1   34.91.13.101
k8s-1-graph-name-neo4j-discovery-public-2   35.204.130.255
```

In `k8s-2-neo4j-main.yaml` replace the IP Addresses in the `k8s-2-graph-name-neo4j-core` StatefulSet spec.template.spec.hostAliases with the IP addresses for the corresponding load balancers from Kubernetes Cluster 1.

#### K8s Cluster 2:

```
$ kubectl get svc -l neo4j.com/role=external-cluster-members -o custom-columns=NAME:.metadata.name,EXTERNAL-IP:..ingress..ip
NAME                                        EXTERNAL-IP
k8s-2-graph-name-neo4j-discovery-public-0   35.246.29.75
k8s-2-graph-name-neo4j-discovery-public-1   34.89.109.124
```

In `k8s-1-neo4j-main.yaml` replace the IP Addresses in the `k8s-1-graph-name-neo4j-core` StatefulSet spec.template.spec.hostAliases with the IP addresses for the corresponding load balancers from Kubernetes Cluster 2.


### Step 3 - Start the Neo4j cluster

First start Neo4j in Kubernetes Cluster 1 (k8s-1). This has 3 Neo4j core members so can form a functioning Neo4j cluster independently of Kubernetes Cluster 2 (k8s-2). 

Once we have a formed Neo4j cluster in k8s-1 we will start the Neo4j core members in k8s-2 which will join the already-running Neo4j cluster.      

#### K8s Cluster 1:

```
kubectl apply -f common-neo4j.yaml -f k8s-1-neo4j-main.yaml -f bolt-addresses-placeholder.yaml
```

The neo4j pod logs can be viewed using kubectl.

```
$ kubectl logs k8s-1-graph-name-neo4j-core-0
...
2020-04-27 15:05:58.780+0000 INFO  Bolt enabled on 0.0.0.0:7687.
2020-04-27 15:06:00.065+0000 INFO  Started.
```

Within a few minutes the logs should show that a cluster has formed and bolt and http remote interfaces have started up. Kubernetes should show the pods as running:

```
$ kubectl get pods -l app.kubernetes.io/name=neo4j
k8s-1-graph-name-neo4j-core-0   1/1     Running   0          6h59m
k8s-1-graph-name-neo4j-core-1   1/1     Running   0          6h57m
k8s-1-graph-name-neo4j-core-2   1/1     Running   0          6h55m
```

n.b. if pulling the neo4j docker images on to your nodes is slow then it may take longer for your cluster to form and some pods might show a some restarts. 

#### K8s Cluster 2:

Once the Neo4j Cluster is formed in k8s-1 we can start 2 more members in k8s-2:

```
# Don't forget to switch clusters e.g. using:
# kubectl config use-context k8s-cluster-2
kubectl apply -f common-neo4j.yaml -f k8s-2-neo4j-main.yaml -f bolt-addresses-placeholder.yaml
```

The neo4j pod logs can be viewed using kubectl as before.

```
$ kubectl logs k8s-2-graph-name-neo4j-core-0
...
2020-04-27 15:05:58.780+0000 INFO  Bolt enabled on 0.0.0.0:7687.
2020-04-27 15:06:00.065+0000 INFO  Started.
```

Within a few minutes the logs should show that the new members have joined the cluster and their bolt and http remote interfaces have started up.

```
$ kubectl get pods -l app.kubernetes.io/name=neo4j
k8s-2-graph-name-neo4j-core-0   1/1     Running   0          6h59m
k8s-2-graph-name-neo4j-core-1   1/1     Running   0          6h57m
k8s-2-graph-name-neo4j-core-2   1/1     Running   0          6h55m
```

If this doesn't happen check that the `hostAliases` were correctly configured in the last step and that the IP addresses of k8s-2 load balancers are accessible from the neo4j pods in cluster 1 (and vice-versa).

You can check that the address is resolving and the port is accessible using these commands:

```
# To check core 0 in k8s-1 can address core 0 in k8s-2
# kubectl config use-context k8s-cluster-1
kubectl exec k8s-1-graph-name-neo4j-core-0 -- bash -c '(echo >/dev/tcp/k8s-2-graph-name-neo4j-discovery-0.default.svc.cluster.local/5000) && echo "success" || echo "failure"'

# To check core 0 in k8s-2 can address core 0 in k8s-1
# kubectl config use-context k8s-cluster-2
kubectl exec k8s-2-graph-name-neo4j-core-0 -- bash -c '(echo >/dev/tcp/k8s-1-grap-name-neo4j-discovery-0.default.svc.cluster.local/5000) && echo "success" || echo "failure"'
```

Congratulations! You should now have a single Neo4j Causal Cluster with members spread across 2 Kubernetes Clusters. However we still have some work to do to be able to use it with Neo4j Browser / Neo4j drivers. 

TODO: test neo4j using `kubectl exec` to run `cypher-shell` inside the K8s cluster.

### Step 4 - Create load balancers for bolt (and http(s))

To access bolt (and the Neo4j browser) from outside the kubernetes clusters we need to set up some more load balancers! 

Once again we will create the load balancers first and then fetch their external IPs.

n.b. depending on your infrastructure provider / configuration you may need to modify these to specify a static external IP address.

#### Kubernetes Cluster 1:
```
kubectl apply -f k8s-1-bolt-load-balancers.yaml
```

#### Kubernetes Cluster 2:
```
kubectl apply -f k8s-2-bolt-load-balancers.yaml
```

### Step 5 - Configure bolt (and http(s)) advertised addresses

We need to get the external IP addresses of the load balancers and use them to fill in our yaml files.

We will use kubectl to get the External IP address for the discovery load balancers and then update the yaml with the IP addresses.

n.b. This time IP addresses from Cluster 1 go in the configuration (yaml) file for Cluster 1.


#### K8s Cluster 1:

```
$ kubectl get svc -l neo4j.com/role=external-clients -o custom-columns=NAME:.metadata.name,EXTERNAL-IP:..ingress..ip
NAME                                   EXTERNAL-IP
k8s-1-graph-name-neo4j-bolt-public-0   34.91.174.200
k8s-1-graph-name-neo4j-bolt-public-1   34.91.13.101
k8s-1-graph-name-neo4j-bolt-public-2   35.204.130.255
```

In `k8s-1-neo4j-bolt-addresses.yaml` the replace IP Addresses in the `graph-name-neo4j-core-address-config` ConfigMap with the IP addresses for the corresponding load balancers from k8s-1.

#### K8s Cluster 2:

```
$ kubectl get svc -l neo4j.com/role=external-clients -o custom-columns=NAME:.metadata.name,EXTERNAL-IP:..ingress..ip
NAME                                   EXTERNAL-IP
k8s-2-graph-name-neo4j-bolt-public-0   35.246.29.75
k8s-2-graph-name-neo4j-bolt-public-1   34.89.109.124
```

In `k8s-2-neo4j-bolt-addresses.yaml` the replace IP Addresses in the `graph-name-neo4j-core-address-config` ConfigMap with the IP addresses for the corresponding load balancers from k8s-2.

### Step 6 - Deploy the bolt (and http(s)) config and restart the pods

#### K8s Cluster 1:

```
kubectl apply -f k8s-1-neo4j-bolt-addresses.yaml
kubectl rollout restart statefulset/k8s-1-graph-name-neo4j-core
# This next command may take a few minutes
kubectl rollout status statefulset/k8s-1-graph-name-neo4j-core
```

#### K8s Cluster 2:

```
kubectl apply -f k8s-2-neo4j-bolt-addresses.yaml
kubectl rollout restart statefulset/k8s-2-graph-name-neo4j-core
# This next command may take a few minutes
kubectl rollout status statefulset/k8s-2-graph-name-neo4j-core
```

You should now be able to log in to your cluster by visiting http://<any bolt load balancer IP>:7474/browser.

n.b. the initial neo4j user password configured in the attached yaml is `qPxGGU3Ouo`.

# TODOs

- set up DNS with A records for all the external bolt IPs.

- use a http ingress or single Load Balancer for http(s) (Neo4j Browser).

- set up encryption for intra-cluster comms (discovery etc.).

- set up encryption for bolt and https (Neo4j Browser).

- use persistent volumes for data dir (trivial)