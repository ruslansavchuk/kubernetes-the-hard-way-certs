# Scheduler

In this section we will configure kube-scheduler to communicate with api-server using the certificate signed with the api-key CA file. and use api-server CA file to validate the server certificate.

## Generate certificates

On the first step, we need to generate scheduler client certificate.

```bash
{
cat > kube-scheduler-csr.json <<EOF
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:kube-scheduler",
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
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
}
```

Generated files:

```
kube-scheduler.csr
kube-scheduler-csr.json
kube-scheduler-key.pem
kube-scheduler.pem
```

## Configure

When certificates are ready, we need to create configuration file which will be used by scheduler to properly configure communication with the api-server. For this we use kubectl

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=kubernetes-ca-api.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

  kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
}
```

Now, we will download and install scheduler binaries

```bash
{
  wget -q --show-progress --https-only --timestamping "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-scheduler"
  chmod +x kube-scheduler
  sudo mv kube-scheduler /usr/local/bin/
}
```

When scheduler binaries are ready, we can move all required configuration files to the proper folder

```bash
sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Create scheduler configuration file

```bash
cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

Create systemd unit file for scheduler service

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
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
  sudo systemctl enable kube-scheduler
  sudo systemctl start kube-scheduler
}
```

And ensure if it operate as expected

```bash
sudo systemctl status kube-scheduler
```

Output:

```
● kube-scheduler.service - Kubernetes Scheduler
     Loaded: loaded (/etc/systemd/system/kube-scheduler.service; enabled; vendor preset: enabled)
     Active: active (running) since Thu 2023-04-20 11:57:44 UTC; 16s ago
       Docs: https://github.com/kubernetes/kubernetes
   Main PID: 15134 (kube-scheduler)
      Tasks: 7 (limit: 2275)
     Memory: 13.7M
     CGroup: /system.slice/kube-scheduler.service
             └─15134 /usr/local/bin/kube-scheduler --config=/etc/kubernetes/config/kube-scheduler.yaml --v=2
...
```

## Verify

In previuos sections we saw that pods created by controller manager are in pending state. Now, if we list the pods created in the cluster, we should see all pods in running state. That is the prove that scheduler operates as expected.

```bash
kubectl get pod --kubeconfig=admin.kubeconfig
```

Result:

```
NAME                    READY   STATUS    RESTARTS   AGE
test-5f6778868d-459xj   1/1     Running   0          12m
```

## Summary

In this section, we configured scheduler to communicate with api-server. The configured scheduler uses certifiate signed by api-server CA. Also, our scheduler is configured to validate server certificate using api-server CA file.

Next: [Kubeproxy](07-kubeproxy.md)