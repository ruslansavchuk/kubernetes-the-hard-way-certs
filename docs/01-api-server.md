# API server

In contrast to ETCD, we will only configure a single instance of the API server. The rationale behind this decision is that setting up multiple instances of the API server does not provide any significant insights regarding the certificates used in the Kubernetes. However, it entails maintaining more files and may confuse the reader.

Before we begin, lets figure out what certificates we need to configure.
1. Client certificate for communication for ETCD cluster (we generated this certificate on the previous step, now we need to configure api-server to use already existing certificate)
2. Server certificate for api-server for communication with api-server clients. We will generate CA certificate, and configure api-server (and clients in future) to trust only certificates signed using out CA.
3. Client certificate for communication with kubelet. This certificate will be signed with different CA certificate, which will be generated specially for communication with kubelet.

## Generate certificates

We will start with the certificates for communication between api-server and clients.
Before we can generate and sign certificates, we need to generate CA.

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
kubernetes-ca-api.csr
kubernetes-ca-api-csr.json
kubernetes-ca-api-key.pem
kubernetes-ca-api.pem
```

Now, we will create the configuration file for signing api-server certificates.

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

And now, we can generate "api-server" server certificate for communication with clients.

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

Generated files:

```
kubernetes-api-server.csr
kubernetes-api-server-csr.json
kubernetes-api-server-key.pem
kubernetes-api-server.pem
```

Now, we will proceed with the certificates required for communication with kubelet.
As previously mentioned, we will use separate CA for this king of communication.
So, we will generate CA.

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
kubernetes-ca-kubelet.csr
kubernetes-ca-kubelet-csr.json
kubernetes-ca-kubelet-key.pem
kubernetes-ca-kubelet.pem
```

Now, create signing config, as we need to sign api-server certificate which will be used as client certificate for communication with kubelet server.

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

When signing config created, we can create api-server -> kubelet client certificate.

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

Generated files:

```
kubernetes-kubelet-server.csr
kubernetes-kubelet-server-csr.json
kubernetes-kubelet-server-key.pem
kubernetes-kubelet-server.pem
```

In addition to already mentioned certificated we need to generate several more things. 
First of all, service account token signing certificate. (todo: we need to add reference what is it or explain)

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

Generated file:

```
service-account.csr
service-account-csr.json
service-account-key.pem
service-account.pem
```

Second, certificate for communication with our api-server. We will configure kubectl to uses this certificate.

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

Generated files:

```
admin.csr
admin-csr.json
admin-key.pem
admin.pem
```

And the last one - encryption key.

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

Now, when all the certificates and secrets generated, we can start configuring api-server.

First of all we need to create folder where we will store kubernetes configuration files.

```bash
sudo mkdir -p /etc/kubernetes/config
```

Next step - download and install API server binary.

```bash
{
  wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-apiserver"
  
  chmod +x kube-apiserver
  sudo mv kube-apiserver /usr/local/bin/
}
```

When binaries installed, we can copy all the certificates to the proper folder.

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

And finally we can create systemd unit file to configue our API server service.

```bash
{
ADD_IP=$(ip a | awk '/inet / && $2 !~ /^127\./ {print $2}' | cut -d '/' -f 1)

cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${ADD_IP} \\
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
}
```

> --service-account-key-file - дуже цікава опція, потрібно для того щоб перевіряти токени які видає сам кубернетес для того щоб прокидати їх у поди для того щоб вони в подальшому могли ті токени викоритовувати (поки не особо ясно яким ca сертифікатом підписувати цей сертифікат)
> 
> The instance internal IP address will be used to advertise the API Server to members of the cluster. 

After systemd unit file created, we need to start it.

```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable kube-apiserver
  sudo systemctl start kube-apiserver
}
```

And check api-server service status.

```bash
systemctl status kube-apiserver
```

Output:

```
● kube-apiserver.service - Kubernetes API Server
     Loaded: loaded (/etc/systemd/system/kube-apiserver.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2024-02-20 14:13:53 UTC; 19s ago
       Docs: https://github.com/kubernetes/kubernetes
   Main PID: 1420 (kube-apiserver)
      Tasks: 10 (limit: 2260)
     Memory: 364.8M
     CGroup: /system.slice/kube-apiserver.service
...
```

# Verify

To verify that api-server works properly we need to download kubectl and ensure that we can communicate with api-server.

```bash
{
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl
  chmod +x kubectl
  sudo mv kubectl /usr/local/bin/
}
```

Now, we need to create kubectl configuration file

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

Also, we will create the deployment. It will help us in the next lessons to ensure that everything works as expected.

```bash
kubectl create deployment test --image=nginx --kubeconfig=admin.kubeconfig
```

And ensure that deployment created.

```bash
kubectl get deployment --kubeconfig=admin.kubeconfig
```

Output:

```
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
test   0/1     0            0           44s
```

## Summary

In this section, we configured api-server to use.
1. Client certificate for communication for ETCD cluster. The authority of the ETCD certificates verified with the usage of etcd CA certificates.
2. Server certificate for api-server for communication with api-server clients. This certificate signed with the usage of separate CA file, and all api-server client also need to sign their certificates using the same CA file, otherwise api-server will not access request from the client.
3. Client certificate for communication with kubelet. This certificate signed with different CA certificate, which is generated specially for communication with kubelet.
4. Service account token signing certificate. This certificate will be used to sign the tokens generated for service accounts.

Next:  [Kubelet](02-kubelet.md)