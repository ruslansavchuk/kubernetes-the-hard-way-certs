# Kubelet

In this section we will configure kubelet to communicate with api-server. 

From previous section reader may remember that we use separate CA for validating certificates:
1. when api-server communicate with kubelet 
2. when kubelet communicate with api-server

But before we can start configuring aspects related to api-sever to kubelet communication, we need to configured services required by kubelet to be able to schedule pods.

## Services required by kubelet

### runc

First service we will configure - [runc](https://github.com/opencontainers/runc).
As this tutorial inspired by KubernetesTheHard way, we will install runc from binaries.
So we need to download and install it.

```bash
{
    wget -q --show-progress --https-only --timestamping https://github.com/opencontainers/runc/releases/download/v1.0.0-rc93/runc.amd64
    sudo mv runc.amd64 runc
    chmod +x runc 
    sudo mv runc /usr/local/bin/
}
```

### containerd

While runc is a lightweight tool for spawning and running containers according to the OCI specification, containerd is a higher-level container runtime that manages the complete container lifecycle. Kubeley uses containerd (in our case) to run pods in Kubernetes cluster.

So, lets download and install it.

```bash
{
    wget -q --show-progress --https-only --timestamping https://github.com/containerd/containerd/releases/download/v1.4.4/containerd-1.4.4-linux-amd64.tar.gz
    mkdir containerd
    tar -xvf containerd-1.4.4-linux-amd64.tar.gz -C containerd
    sudo mv containerd/bin/* /bin/
}
```

After we donwnload and install containerd binaries, we need to configure containerd to use runc as tool to run containers.

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

When containerd configuration is ready, we can create systemd unit file for containerd service

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

Run it

```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd
  sudo systemctl start containerd
}
```

And check service status

```bash
systemctl status containerd
```

Output:

```
● containerd.service - containerd container runtime
     Loaded: loaded (/etc/systemd/system/containerd.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2024-02-20 21:21:56 UTC; 11s ago
       Docs: https://containerd.io
    Process: 2211 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
   Main PID: 2226 (containerd)
      Tasks: 10 (limit: 2260)
     Memory: 22.0M
     CGroup: /system.slice/containerd.service
             └─2226 /bin/containerd
...
```
### cni

Now, our kubelet can run containers if container uses host network. But to make kubelet abble to run container "with its own network interface", we need to configure [cni](https://github.com/containernetworking/cni).

As previously, we will install it from binaries, so, lets download binaries.

```bash
{
    wget -q --show-progress --https-only --timestamping https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz
    sudo mkdir -p /etc/cni/net.d /opt/cni/bin
    sudo tar -xvf cni-plugins-linux-amd64-v0.9.1.tgz -C /opt/cni/bin/
}
```

When binaries are downloaded, we can create configuration file for the CNI.

```bash
{
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

cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.4.0",
    "name": "lo",
    "type": "loopback"
}
EOF
}
```

So, all prerequired steps completed.

## kubelet

As previously, we need to download kubelet binaries

```bash
{
wget -q --show-progress --https-only --timestamping https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubelet
chmod +x kubelet
sudo mv kubelet /usr/local/bin/
}
```

Now, when we have all required binaries, we can start generating certificates required by kubelets.

First certificate we will generate - service certificate for kubelet

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

Generated files:

```
kubelet-from-api-server.csr
kubelet-from-api-server-csr.json
kubelet-from-api-server-key.pem
kubelet-from-api-server.pem
```

As you can see, during certificate configuration, we added HOST_NAME variable to certificate host names. We added it, because in future, Kubernetes will use node host name as the host for communication, and during this process certificate host should be validated.

Second think we are goinf to generate - client certificate for communication with the api-server

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

Generated files:
```
kubelet-to-api-server.csr
kubelet-to-api-server-csr.json
kubelet-to-api-server-key.pem
kubelet-to-api-server.pem
```

In this certificate, the CN field specially formatted, based on this filed, api-server will authorize the client who uses this certificate as client from "node group" with the name "${HOST_NAME}". 

After certificate generated, we need to create the configuration file for kubelet and say how kubelet should communicate to the api-server. We will configure it using kubectl. 

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

> Note:
> In the script above, we build configuration file where:
> - --certificate-authority - certificate which will be used to validate api-server certificate
> - --client-(certificate/key) - certificate/key which will be used during communication with the api-server, also used for authentication

After the file successfully created, we need to distribute kubelets certificates to the proper folder

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

Create kubelet configuration file

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

>Important configuration options:
>authentication.x509.clientCAFile - the path to the ca certificate file which will be used to validate client certificate file, when specified, says kubelet that client certificate can be used to authorize the client 
>tlsCertFile/tlsPrivateKeyFile - path to the certificate/key file which will be used by kubelet server during communication with the clients

And create systemd unit file for kubelet service

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

After all the configuration files are ready, we can start kubelet daemon

```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable kubelet
  sudo systemctl restart kubelet
}
```

And check it's status

```bash
sudo systemctl status kubelet
```

Output:

```
● kubelet.service - Kubernetes Kubelet
     Loaded: loaded (/etc/systemd/system/kubelet.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2024-02-20 21:25:23 UTC; 6s ago
       Docs: https://github.com/kubernetes/kubernetes
   Main PID: 2403 (kubelet)
      Tasks: 12 (limit: 2260)
     Memory: 34.7M
     CGroup: /system.slice/kubelet.service
...
```

## Verify

When our kubelet service is ready, we can check if it successfully established the connection with the api-server.
To do that, we wiil retriew the list of nodes attached to our cluster.

```bash
kubectl get nodes --kubeconfig=admin.kubeconfig
```

If everything is success we should receive similar output

```
NAME             STATUS   ROLES    AGE   VERSION
example-server   Ready    <none>   28s   v1.21.0
```

> if your node is in "NotReady" state, wait a bit, and only after that, if status will not change to "Ready" start investigating kubelet log.
> kubletet need some time to perform all the required checks and ansure that it is ready to run pods.

Also, in addition to already configured things for kubelet, we need to create cluster rile binding. This cluster role binding is required during the pod logs retriewal (we will ensure that log retrieval process works as expected in on of the next sessionc). 

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

## Summary

In this section, we added "worker node" to our kubernetes cluster which can run pods. On this node, we configured kubelet to use two different certificates:
1. for communication with api-server
2. for accepting requests from api-server
Both certificates are signed with the usage of different CA certificates, and during communication, kubelets ensure that it communicate with the client/server which use certificates signed by the proper CA certificate.

Next: [Controller manager](03-controller-manager.md)
