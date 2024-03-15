# Installing the Client Tools

##### In this lab we will install the command line utilites required to complete the setup:
- cfssl
- cfssljson
- kubectl

### We will be using a jump server inside the VPC we will create further to install these requirements, so that we can easily transfer the files in the kube-nodes

## 1. Install CFSSL

The cfssl and cfssljson command line utilities will be used to provision a PKI Infrastructure and generate TLS certificates.

#### Linux

    wget -q --show-progress --https-only --timestamping \
    https://github.com/cloudflare/cfssl/releases/download/v1.6.0/cfssl_1.6.0_linux_amd64  -O cfssl
---
    wget -q --show-progress --https-only --timestamping \
    https://github.com/cloudflare/cfssl/releases/download/v1.6.0/cfssljson_1.6.0_linux_amd64 -O cfssljson




    chmod +x cfssl cfssljson

    sudo mv cfssl cfssljson /usr/local/bin/

### Verification
Verify cfssl and cfssljson version, must higher than 1.4.1:

    cfssl version

    cfssljson --version

## 2. Install kubectl

Kubernetes provides a command line tool for communicating with a Kubernetes cluster's control plane, using the Kubernetes API.

#### Linux

    wget https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl

    chmod +x kubectl

    sudo mv kubectl /usr/local/bin/

### Verify

    kubectl version --client