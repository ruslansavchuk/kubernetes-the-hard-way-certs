## runc

Lets download runc binaries

```bash
wget -q --show-progress --https-only --timestamping \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc93/runc.amd64
```

As download process complete, we need to move runc binaries to bin folder

```bash
{
    sudo mv runc.amd64 runc
    chmod +x runc 
    sudo mv runc /usr/local/bin/
}
```

## containerd

As already mentioned, Container Runtime: The software responsible for running and managing containers on the worker nodes. The container runtime is responsible for pulling images, creating containers, and managing container lifecycles

Now, let's download containerd.

```bash
wget -q --show-progress --https-only --timestamping \
  https://github.com/containerd/containerd/releases/download/v1.4.4/containerd-1.4.4-linux-amd64.tar.gz
```

Unzip containerd binaries to the bin directory

```bash
{
    mkdir containerd
    tar -xvf containerd-1.4.4-linux-amd64.tar.gz -C containerd
    sudo mv containerd/bin/* /bin/
}
```

In comparison to the runc, containerd is a service which can be called by someone to run container.

Before we will run containerd service, we need to configure it.
```bash
{
sudo mkdir -p /etc/containerd/

cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF
}
```

As we can see, we configured containerd to use runc (we installed before) to run containers.

Now we can configure contanerd service
```bash
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
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

Run service
```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd
  sudo systemctl start containerd
}
```

# cni

```bash
wget -q --show-progress --https-only --timestamping \
  https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz
```

```bash
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin
```

```bash
sudo tar -xvf cni-plugins-linux-amd64-v0.9.1.tgz -C /opt/cni/bin/
```

```bash
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
          [{"subnet": "10.240.1.0/24"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

```bash
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.4.0",
    "name": "lo",
    "type": "loopback"
}
EOF
```


# kubelet
для початку його потрібно завантажити
```bash
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubelet
```

```bash
{
  chmod +x kubelet 
  sudo mv kubelet /usr/local/bin/
}
```

# todo - merge
# Kubelet

```bash
{
HOST_NAME=$(hostname -a)
cat > kubelet-to-api-server-csr.json <<EOF
{
  "CN": "system:node:${HOST_NAME}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=kubernetes-ca-api.pem \
  -ca-key=kubernetes-ca-api-key.pem \
  -config=kubernetes-ca-api-config.json \
  -hostname=127.0.0.1 \
  -profile=kubernetes-api \
  kubelet-to-api-server-csr.json | cfssljson -bare kubelet-to-api-server
}
```

```bash
{
HOST_NAME=$(hostname -a)
cat > kubelet-from-api-server-csr.json <<EOF
{
  "CN": "system:node:${HOST_NAME}",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:nodes",
      "OU": "Kubernetes The Hard Way",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert \
  -ca=kubernetes-ca-kubelet.pem \
  -ca-key=kubernetes-ca-kubelet-key.pem \
  -config=kubernetes-ca-kubelet-config.json \
  -hostname=127.0.0.1,${HOST_NAME} \
  -profile=kubernetes-kubelet \
  kubelet-from-api-server-csr.json | cfssljson -bare kubelet-from-api-server
}
```

це конфігураційний файл кублєта - розказує кублєту як ходити на апі сервер
```bash
{
HOST_NAME=$(hostname -a)
kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=kubernetes-ca-api.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kubelet.kubeconfig

kubectl config set-credentials system:node:${HOST_NAME} \
    --client-certificate=kubelet-to-api-server.pem \
    --client-key=kubelet-to-api-server-key.pem \
    --embed-certs=true \
    --kubeconfig=kubelet.kubeconfig

kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${HOST_NAME} \
    --kubeconfig=kubelet.kubeconfig

kubectl config use-context default --kubeconfig=kubelet.kubeconfig
}
```

```bash
{
  sudo mkdir /var/lib/kubelet/
  
  sudo cp \
    kubelet-to-api-server-key.pem kubelet-to-api-server.pem \
    kubelet-from-api-server-key.pem kubelet-from-api-server.pem \
    /var/lib/kubelet/

  sudo cp \
    kubernetes-ca-api.pem \
    kubernetes-ca-kubelet.pem \
    /var/lib/kubernetes/
    
  sudo cp kubelet.kubeconfig /var/lib/kubelet/kubeconfig
}
```

```bash
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/kubernetes-ca-kubelet.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "10.240.1.0/24"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/kubelet-from-api-server.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/kubelet-from-api-server-key.pem"
EOF
```

```bash
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable kubelet
  sudo systemctl restart kubelet
}
```

```bash
sudo systemctl status kubelet
```

такс, ну і тепер момент істини, потрібно розібратись чи з'явились у нас ноди у кластері

```bash
kubectl get nodes --kubeconfig=admin.kubeconfig
```

```bash
{
cat <<EOF> node-auth.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-proxy-access-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kubelet-api-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: kubernetes
EOF

kubectl apply -f node-auth.yml --kubeconfig=admin.kubeconfig
}
```

Next: [Controller manager](04-controller-manager.md)
