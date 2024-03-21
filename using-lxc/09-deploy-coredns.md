# Deploying the DNS Cluster Add-on 
CoreDNS can operate in the Kubernetes cluster with standard Kube-DNS. As a plug of Kubernetes, CoreDNS will read the area （zone） data from the Kubernetes cluster. It realizes the specification defined for the DNS service discovery of Kubernetes：[Kubernetes DNS-Based Service Discovery](https://github.com/kubernetes/dns/blob/master/docs/specification.md).
## The DNS Cluster Add-on
Deploy the coredns cluster add-on:
```
kubectl apply -f https://storage.googleapis.com/kubernetes-the-hard-way/coredns.yaml
```