# kube-proxy

In this section we will configure kube-proxt to communicate with api-server using the certificate signed with the api-key CA file. and use api-server CA file to validate the server certificate.

## Generate certificates

On the first step, we need to generate kube-proxy client certificate.

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

## Configure

When certificates are ready, we need to create configuration file which will be used by kube-proxy to properly configure communication with the api-server. For this we use kubectl

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

Now, we will download and install kube-proxy binaries

```bash
{
    wget -q --show-progress --https-only --timestamping https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-proxy
    sudo mkdir -p /var/lib/kube-proxy
    chmod +x kube-proxy
    sudo mv kube-proxy /usr/local/bin/
}
```

When kube-proxy binaries are ready, we can move all required configuration files to the proper folder

```bash
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Create kube-proxy configuration file

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

Create systemd unit file for kube-proxy service

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

Start scheduler service

```bash
{
    sudo systemctl daemon-reload
    sudo systemctl enable kube-proxy
    sudo systemctl start kube-proxy
}
```

And ensure if it operate as expected

```bash
sudo systemctl status kube-proxy
```

Output:

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

## Verify

The best way to ensure that kubeproxy works as expected - create service and check if it is awailable.

So, lets create service

```bash
kubectl create svc clusterip test --tcp=80:80 --kubeconfig=admin.kubeconfig
```

And try to connect to the service connect to the service created

```bash
kubectl run test-connectivity --kubeconfig=admin.kubeconfig -i --rm --image=busybox --restart=Never -- wget -O - $(kubectl get svc test -o=jsonpath='{.spec.clusterIP}' --kubeconfig=admin.kubeconfig)
```

Output:

```
Connecting to 10.32.0.113 (10.32.0.113:80)
writing to stdout
-                    100% |********************************|   615  0:00:00 ETA
written to stdout
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
pod "test-connectivity" deleted
```

## Summary

In this section, we configured kube-proxy to communicate with api-server. The configured kube-proxy uses certifiate signed by api-server CA. Also, our scheduler is configured to validate server certificate using api-server CA file.
