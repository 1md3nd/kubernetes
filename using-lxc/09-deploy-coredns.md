# Deploying the DNS Cluster Add-on 
In this we will deploy the DNS add-on which provides DNS based service discovery, backed by CoreDNS, to applications running inside the Kubernetes cluster.

## The DNS Cluster Add-on
Deploy the coredns cluster add-on:
```
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml
```