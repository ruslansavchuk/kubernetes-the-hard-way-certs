```bash
{
cat > kube-proxy-csr.json <<EOF
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-proxy",
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
  -profile=kubernetes-api \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
}
```

Generated files:

```
kube-proxy.csr
kube-proxy-csr.json
kube-proxy-key.pem
kube-proxy.pem
```

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=kubernetes-ca-api.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-credentials system:kube-proxy \
    --client-certificate=kube-proxy.pem \
    --client-key=kube-proxy-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-proxy \
    --kubeconfig=kube-proxy.kubeconfig

  kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
}
```

```bash
{
    wget -q --show-progress --https-only --timestamping https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-proxy
    sudo mkdir -p /var/lib/kube-proxy
    chmod +x kube-proxy
    sudo mv kube-proxy /usr/local/bin/
}
```

```bash
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

```bash
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
    kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

```bash
{
    sudo systemctl daemon-reload
    sudo systemctl enable kube-proxy
    sudo systemctl start kube-proxy
}
```

```bash
sudo systemctl status kube-proxy
```

```
● kube-proxy.service - Kubernetes Kube Proxy
     Loaded: loaded (/etc/systemd/system/kube-proxy.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2024-02-20 22:09:59 UTC; 6s ago
       Docs: https://github.com/kubernetes/kubernetes
   Main PID: 7472 (kube-proxy)
      Tasks: 7 (limit: 2260)
     Memory: 12.3M
     CGroup: /system.slice/kube-proxy.service
             └─7472 /usr/local/bin/kube-proxy --config=/var/lib/kube-proxy/kube-proxy-config.yaml
...
```

Next: [Summary](08-conclusion.md)