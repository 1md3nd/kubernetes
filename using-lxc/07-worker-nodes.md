# Bootstrapping the Kubernetes Worker Nodes

Now we are going to setup the worker nodes for that we will be creating these services:
-   kubelet
-   kube-proxy
-   containerd

## Download and Install Worker Binaries
    wget -q --show-progress --https-only --timestamping \
    https://storage.googleapis.com/kubernetes-release/release/v1.29.0/bin/linux/amd64/kubectl \
    https://storage.googleapis.com/kubernetes-release/release/v1.29.0/bin/linux/amd64/kube-proxy \
    https://storage.googleapis.com/kubernetes-release/release/v1.29.0/bin/linux/amd64/kubelet

Installation directories:

    sudo mkdir -p \
    /etc/cni/net.d \
    /opt/cni/bin \
    /var/lib/kubelet \
    /var/lib/kube-proxy \
    /var/lib/kubernetes \
    /var/run/kubernetes


Installing work binaries:
```
chmod +x kubectl kube-proxy kubelet 
sudo mv kubectl kube-proxy kubelet /usr/local/bin/
```

```
{
wget https://github.com/containerd/containerd/releases/download/v1.6.8/containerd-1.6.8-linux-amd64.tar.gz

wget https://github.com/opencontainers/runc/releases/download/v1.1.3/runc.amd64


sudo tar Cxzvf /usr/local containerd-1.6.8-linux-amd64.tar.gz
sudo install -m 755 runc.amd64 /usr/local/sbin/runc

# Now, we need the Container Network Interface, which is used to provide the necessary networking functionality. Download CNI with:

wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz


sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz

sudo mkdir /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml

sudo curl -L https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -o /etc/systemd/system/containerd.service

}
```
## Configure CNI Networking
Retrieve the Pod CIDR range for the current container:

    - worker-0
    POD_CIDR=10.200.0.0/24
    - worker-1
    POD_CIDR=10.200.1.0/24
    - worker-2
    POD_CIDR=10.200.2.0/24

    echo "${POD_CIDR}"


Create the bridge network configuration file:

    cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
    {
        "cniVersion": "0.4.0",
        "name": "bridge",
        "type": "bridge",
        "bridge": "cnio0",
        "isGateway": true,
        "ipMasq": true,
        "ipam": {
            "type": "host-local",
            "ranges": [
            [{"subnet": "${POD_CIDR}"}]
            ],
            "routes": [{"dst": "0.0.0.0/0"}]
        }
    }
    EOF


Create the loopback network configuration file:

    cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
    {
        "cniVersion": "0.4.0",
        "name": "lo",
        "type": "loopback"
    }
    EOF


## Configure containerd
Create the containerd configuration file:
```
sudo mkdir -p /etc/containerd/
```
```
- v 1.7
- DONT RUN THIS COMMAND

/etc/containerd/config.toml

version = 2

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true

[plugins."io.containerd.grpc.v1.cri".containerd]
  snapshotter = "overlayfs"
  [plugins."io.containerd.grpc.v1.cri".containerd.default_runtime]
    runtime_type = "io.containerd.runc.v2"  # Updated to use runtime v2
    runtime_engine = "/usr/local/bin/runc"
    runtime_root = ""

[plugins."io.containerd.grpc.v1.cri".containerd.cgroup]
  v2 = true  # Removed deprecated 'path' and 'default_slice' settings
EOF
```
> DONT RUN THIS COMMAND

Edit the containerd.service systemd unit file:
```
nano /etc/systemd/system/containerd.service
```
REMOVE LINE 
```
ExecStartPre=/sbin/modprobe overlay
```
It should look like this 
```
/etc/systemd/system/containerd.service

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

## Configure the Kubelet
    
    WORKER_NAME=$(hostname -s)
    echo "${WORKER_NAME}"

    sudo mv ${WORKER_NAME}-key.pem ${WORKER_NAME}.pem /var/lib/kubelet/
    sudo mv ${WORKER_NAME}.kubeconfig /var/lib/kubelet/kubeconfig
    sudo mv ca.pem /var/lib/kubernetes/


Create the kubelet-config.yaml configuration file:

```
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: "systemd"
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${WORKER_NAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${WORKER_NAME}-key.pem"
EOF
```

Create the kubelet.service systemd unit file:cd
```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --register-node=true \\
  --fail-swap-on=false \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

<!-- --cpu-manager-policy=static \\
    --kube-reserved=cpu=1,memory=2Gi,ephemeral-storage=1Gi \\
    --system-reserved=cpu=1,memory=2Gi,ephemeral-storage=1Gi \\ -->
<!-- ---
    systemctl daemon-reload
    systemctl restart kubelet -->
## Configure the Kubernetes Proxy

    sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig

Create the kube-proxy-config.yaml configuration file:
```
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```
Create the kube-proxy.service systemd unit file:
```
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
--config=/var/lib/kube-proxy/kube-proxy-config.yaml \\
--conntrack-max-per-core=0
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
```
systemctl daemon-reload
sudo systemctl enable --now containerd kubelet kube-proxy
systemctl restart containerd kubelet kube-proxy
```
