# API server

something about api server configuration

## Generate certificates

### API-server-to-client

#### Generate ca certificate
```bash
{
cat > kubernetes-ca-api-csr.json <<EOF
{
  "CN": "kubernetes-api",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca kubernetes-ca-api-csr.json | cfssljson -bare kubernetes-ca-api
}
```

Generated files:
```
kubernetes-ca-api-key.pem
kubernetes-ca-api.csr
kubernetes-ca-api.pem
```

Generate signing config
```bash
cat > kubernetes-ca-api-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes-api": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

Generate certificate for communication with the clients
```bash
{
cat > kubernetes-api-server-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
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
  -hostname=control-plane,127.0.0.1 \
  -profile=kubernetes-api \
  kubernetes-api-server-csr.json | cfssljson -bare kubernetes-api-server
}
```

### For communication between API server and kubelet

```bash
{
cat > kubernetes-ca-kubelet-csr.json <<EOF
{
  "CN": "kubernetes-kubelet",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
      "OU": "CA",
      "ST": "Oregon"
    }
  ]
}
EOF

cfssl gencert -initca kubernetes-ca-kubelet-csr.json | cfssljson -bare kubernetes-ca-kubelet
}
```

Generated file:
```
kubernetes-ca-kubelet-key.pem
kubernetes-ca-kubelet.csr
kubernetes-ca-kubelet.pem
```

Generate signing config
```bash
cat > kubernetes-ca-kubelet-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "kubernetes-kubelet": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

For communication with kubelet

```bash
{
cat > kubernetes-kubelet-server-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
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
  -hostname=127.0.0.1 \
  -profile=kubernetes-kubelet \
  kubernetes-kubelet-server-csr.json | cfssljson -bare kubernetes-kubelet-server
}
```

### Generate certificate for signing service account tokens

```bash
{
cat > service-account-csr.json <<EOF
{
  "CN": "service-accounts",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "Kubernetes",
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
  service-account-csr.json | cfssljson -bare service-account
}
```

### Generate certificate for admin to communicate with API server

```bash
{
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "Portland",
      "O": "system:masters",
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
  admin-csr.json | cfssljson -bare admin
}
```

### Generate encryption key

Kubernetes stores a variety of data including cluster state, application configurations, and secrets. Kubernetes supports the ability to [encrypt](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data) cluster data at rest.

In this lab you will generate an encryption key and an [encryption config](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/#understanding-the-encryption-at-rest-configuration) suitable for encrypting Kubernetes Secrets.

Generate an encryption key:

```bash
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```

Create the `encryption-config.yaml` encryption config file:

```bash
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF
```

## Provision the Kubernetes Control Plane

Create the Kubernetes configuration directory:

```bash
sudo mkdir -p /etc/kubernetes/config
```

```bash
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-apiserver"
```
Install the Kubernetes binaries:

```bash
{
  chmod +x kube-apiserver
  sudo mv kube-apiserver /usr/local/bin/
}
```

```bash
{
  sudo mkdir -p /var/lib/kubernetes/

  sudo cp \
    etcd-ca-cluster-to-client.pem \
    encryption-config.yaml \
    kubernetes-ca-api.pem \
    kubernetes-ca-kubelet.pem \
    service-account-key.pem service-account.pem \
    etcd-cluster-to-client-client-key.pem etcd-cluster-to-client-client.pem \
    kubernetes-kubelet-server-key.pem kubernetes-kubelet-server.pem \
    kubernetes-api-server-key.pem kubernetes-api-server.pem \
    /var/lib/kubernetes/
}
```

The instance internal IP address will be used to advertise the API Server to members of the cluster. Retrieve the internal IP address for the current compute instance:

Create the `kube-apiserver.service` systemd unit file:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address='127.0.0.1' \\
  --allow-privileged='true' \\
  --apiserver-count='3' \\
  --audit-log-maxage='30' \\
  --audit-log-maxbackup='3' \\
  --audit-log-maxsize='100' \\
  --audit-log-path='/var/log/audit.log' \\
  --authorization-mode='Node,RBAC' \\
  --bind-address='0.0.0.0' \\
  --client-ca-file='/var/lib/kubernetes/kubernetes-ca-api.pem' \\
  --enable-admission-plugins='NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota' \\
  --etcd-cafile='/var/lib/kubernetes/etcd-ca-cluster-to-client.pem' \\
  --etcd-certfile='/var/lib/kubernetes/etcd-cluster-to-client-client.pem' \\
  --etcd-keyfile='/var/lib/kubernetes/etcd-cluster-to-client-client-key.pem' \\
  --etcd-servers='https://127.0.0.1:2369,https://127.0.0.1:2379,https://127.0.0.1:2389' \\
  --event-ttl='1h' \\
  --encryption-provider-config='/var/lib/kubernetes/encryption-config.yaml' \\
  --kubelet-certificate-authority='/var/lib/kubernetes/kubernetes-ca-kubelet.pem' \\
  --kubelet-client-certificate='/var/lib/kubernetes/kubernetes-kubelet-server.pem' \\
  --kubelet-client-key='/var/lib/kubernetes/kubernetes-kubelet-server-key.pem' \\
  --runtime-config='api/all=true' \\
  --service-cluster-ip-range='10.32.0.0/24' \\
  --service-node-port-range='30000-32767' \\
  --tls-cert-file='/var/lib/kubernetes/kubernetes-api-server.pem' \\
  --tls-private-key-file='/var/lib/kubernetes/kubernetes-api-server-key.pem' \\
  --service-account-key-file='/var/lib/kubernetes/service-account.pem' \\
  --service-account-signing-key-file='/var/lib/kubernetes/service-account-key.pem' \\
  --service-account-issuer='https://kubernetes.default.svc.cluster.local' \\
  --api-audiences='https://kubernetes.default.svc.cluster.local' \\
  --v='2'
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

> --service-account-key-file - дуже цікава опція, потрібно для того щоб перевіряти токени які видає сам кубернетес для того щоб прокидати їх у поди для того щоб вони в подальшому могли ті токени викоритовувати (поки не особо ясно яким ca сертифікатом підписувати цей сертифікат)

```bash
{
  systemctl enable kube-apiserver
  systemctl start kube-apiserver
}
```

```bash
systemctl status kube-apiserver
```

# Verify

Download kubectl binaries
```bash
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl \
  && chmod +x kubectl \
  && sudo mv kubectl /usr/local/bin/
```

Generate kubectl configuration file

```bash
{
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=kubernetes-ca-api.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

  kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

  kubectl config use-context default --kubeconfig=admin.kubeconfig
}
```

Ensure kubectl can communicate with API server

```bash
kubectl version --kubeconfig=admin.kubeconfig
```
Next:  [Kubelet](03-kubelet.md)