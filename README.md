# nokia-nsp-k8s-poc
Nokia NSP - K8s setup for deploying Neo4j Cluster across 2 Kubernetes cluster

Customer: Nokia NSP
Main Purpose: To be able to deploy Neo4j Causal Cluster across 2 Kubernetes Cluster in different regions. This is required to be able to cater to DR. 
Currently trying to achieve with this project and YAML files: Trying to setup Neo4j Causal Cluster on single kubernetes cluster with Neo4j version 3.5.14 - with NodePort mapped to port 5000 and Node IP and configuring that as part of initial_discovery_members.

