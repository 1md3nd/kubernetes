# Kubernetes the hard way v1.29.0

we are going to learn how to setup and use kubernetes


1. Using LXC containers as nodes: [Using LXC](https://github.com/1md3nd/kubernetes/tree/main/using-lxc)
    - We will be using `lxc` containers to launch nodes and setup it.
    - We are using latest version of kubernetes v1.29.0 `Till March 2024`
    - We will create 3 `master(controller)` node 3 `worker nodes` and 1 `haproxy` for laodbalancing.
    - Completed and running.

2. Using AWS EC2 instances as nodes: [Using AWS](https://github.com/1md3nd/kubernetes/tree/main/using-aws)
    - We will use `AWS` to provisioning the compute infrastructure required to bootstrap a Kubernetes cluster from scratch.
    - we will sometimes use `aws cloudshell` to spin some infra and also using AWS Console for some basic purpose.
    - In progress.

