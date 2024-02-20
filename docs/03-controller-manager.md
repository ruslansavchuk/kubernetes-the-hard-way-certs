# Controller manager

In this section we will configure controller manager to communicate with api-server using the certificate signed with the api-key CA file. and use api-server CA file to validate the server certificate.

## Generate certificates

On the first step, we need to generate controller manager client certificate.

```bash
{
cat > kube-controller-manager-csr.json <<EOF
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-controller-manager",
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
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
}
```

Generated files:

```
kube-controller-manager.csr
kube-controller-manager-csr.json
kube-controller-manager-key.pem
kube-controller-manager.pem
```

## Configure

When certificates are ready, we need to create configuration file chich will be used by controller mamanger to properly configure communication with the api-server. For this we use kubectl

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=kubernetes-ca-api.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

  kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
}
```

Now, we will download and install controller manager binaries

```bash
{
  wget -q --show-progress --https-only --timestamping "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-controller-manager"
  chmod +x kube-controller-manager
  sudo mv kube-controller-manager /usr/local/bin/
}
```

When controller manager binaries are ready, we can move all required configuration files to the proper folder

```bash
{
sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
sudo cp kubernetes-ca-api.pem kubernetes-ca-api-key.pem /var/lib/kubernetes/
}
```

Create controller manager systemd unit file

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --bind-address=0.0.0.0 \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/kubernetes-ca-api.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/kubernetes-ca-api-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/kubernetes-ca-api.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

> note: 
> service-account-private-key-file - for service account token generation
> root-ca-file - for secret related to service account creation
> cluster-signing-cert-file and cluster-signing-key-file - for certificate signing requests handling

Start controller manager service

```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-controller-manager
  sudo systemctl start kube-controller-manager
}
```

And ensure that it works as expected

```bash
sudo systemctl status kube-controller-manager
```

```
● kube-controller-manager.service - Kubernetes Controller Manager
     Loaded: loaded (/etc/systemd/system/kube-controller-manager.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2023-04-20 11:48:41 UTC; 30s ago
       Docs: https://github.com/kubernetes/kubernetes
   Main PID: 14805 (kube-controller)
      Tasks: 6 (limit: 2275)
     Memory: 32.0M
     CGroup: /system.slice/kube-controller-manager.service
             └─14805 /usr/local/bin/kube-controller-manager --bind-address=0.0.0.0 --cluster-cidr=10.200.0.0/16 --cluster-name=kubernetes --cluster-signing-cert-file=/var/lib/kubernetes/c>
...
```

## Verify

In previuos sections we created the deployment in our kubernetes cluster, but nothing happened after that. The main reason - it is controller manager responsibility to create replica set and pods for the deployment created. After we configured controller manager in our cluster, we can ensure that replica set and pod was created for our deployment.

Check if replicaset created.

```bash
kubectl get replicaset --kubeconfig=admin.kubeconfig
```

Output:

```
NAME              DESIRED   CURRENT   READY   AGE
test-5f6778868d   1         1         0       5m28s
```

Check if pod created

```bash
kubectl get pod --kubeconfig=admin.kubeconfig
```

Output:

```
NAME                    READY   STATUS    RESTARTS   AGE
test-5f6778868d-459xj   0/1     Pending   0          6m12s
```

As you can see, contoller manager works as expected.

## Summary

In this section, we configured contoller manager to communicate with api-server. The configured contoller manager uses certifiate signed by api-server CA. Also, our controller manager validate server certificate using api-server CA file.

Next: [Scheduler](06-scheduler.md)