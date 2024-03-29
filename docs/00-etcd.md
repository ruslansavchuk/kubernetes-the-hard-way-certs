# ETCD

In this segment, we'll establish an ETCD cluster. Our cluster will consist of 3 nodes, with communication between clients and nodes secured through the use of two types of certificates:
- Certificates for inter-node communication within the cluster
- Certificates for communication between the cluster and clients

![image](./img/00-etcd.png "Comminucation schema")

### Cluster configuration

ETCD can operate as a single server or as a cluster. Since our objective is to explore the certificates utilized in a Kubernetes cluster, we will configure ETCD in cluster mode. This approach enables us to observe how communication between ETCD cluster nodes is secured.

## Generate ETCD peer certificates

As already mentioned, we will set up secure communication between cluster nodes. To achieve this, we will generate a certificate authority certificate. This certificate will be utilized for signing the certificates intended for use by ETCD cluster nodes.

To do that, execute

```bash
{
cat > etcd-ca-peer-csr.json <<EOF
{
  "CN": "etcd-peer",
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

cfssl gencert -initca etcd-ca-peer-csr.json | cfssljson -bare etcd-ca-peer
}
```

Generated filed:
```
etcd-ca-peer.csr
etcd-ca-peer-csr.json
etcd-ca-peer-key.pem
etcd-ca-peer.pem
```

Generate config for signing certificates using generated ca certificate
```bash
cat > etcd-ca-peer-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "etcd-peer": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```

Now, after ca and signing config generated, we can generate certificates for communication between cluster nodes
```bash
{
for instance in 1 2 3; do
cat > etcd-peer-${instance}-csr.json <<EOF
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
  -ca=etcd-ca-peer.pem \
  -ca-key=etcd-ca-peer-key.pem \
  -config=etcd-ca-peer-config.json \
  -hostname=127.0.0.1 \
  -profile=etcd-peer \
  etcd-peer-${instance}-csr.json | cfssljson -bare etcd-peer-${instance}
done
}
```

Generated files:
```
etcd-peer-1.csr
etcd-peer-1-csr.json
etcd-peer-1-key.pem
etcd-peer-1.pem
etcd-peer-2.csr
etcd-peer-2-csr.json
etcd-peer-2-key.pem
etcd-peer-2.pem
etcd-peer-3.csr
etcd-peer-3-csr.json
etcd-peer-3-key.pem
etcd-peer-3.pem
```

## Generate ETCD client certificates

To configure secure communication between ETCD cluster and clients (in our case api-server) we will generate one more ca certificate and certificates for client and etcd server. In etcd this ca and cert file can be different from the certificates used inside etcd cluster.

```bash
{
cat > etcd-ca-cluster-to-client-csr.json <<EOF
{
  "CN": "etcd-cluster-to-client",
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

cfssl gencert -initca etcd-ca-cluster-to-client-csr.json | cfssljson -bare etcd-ca-cluster-to-client
}
```

Generated files:
```
etcd-ca-cluster-to-client-key.pem
etcd-ca-cluster-to-client.csr
etcd-ca-cluster-to-client-csr.json
etcd-ca-cluster-to-client.pem
```

Generate config for signing server-to-client certificates
```bash
cat > etcd-ca-cluster-to-client-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "etcd-cluster-to-client": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF
```


Generate certificate which will be used by the server to communicate with clients

```bash
{
cat > etcd-cluster-to-client-server-csr.json <<EOF
{
  "CN": "etcd-cluster-to-client-server",
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
  -ca=etcd-ca-cluster-to-client.pem \
  -ca-key=etcd-ca-cluster-to-client-key.pem \
  -config=etcd-ca-cluster-to-client-config.json \
  -hostname=127.0.0.1 \
  -profile=etcd-cluster-to-client \
  etcd-cluster-to-client-server-csr.json | cfssljson -bare etcd-cluster-to-client-server
}
```

Generated files:
```
etcd-cluster-to-client-server-key.pem
etcd-cluster-to-client-server.csr
etcd-cluster-to-client-server.pem
```

Similar for the client

```bash
{
cat > etcd-cluster-to-client-client-csr.json <<EOF
{
  "CN": "etcd-cluster-to-client-client",
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
  -ca=etcd-ca-cluster-to-client.pem \
  -ca-key=etcd-ca-cluster-to-client-key.pem \
  -config=etcd-ca-cluster-to-client-config.json \
  -hostname=127.0.0.1 \
  -profile=etcd-cluster-to-client \
  etcd-cluster-to-client-client-csr.json | cfssljson -bare etcd-cluster-to-client-client
}
```

Generated files:
```
etcd-cluster-to-client-client-key.pem
etcd-cluster-to-client-client.csr
etcd-cluster-to-client-client.pem
```

## Configure ETCD cluster

Now, we will set-up ETCD cluster.

First of all, we need to download binaries of our ETCD service.
```bash
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.15/etcd-v3.4.15-linux-amd64.tar.gz"
```

Un-tar and move to the proper directory
```bash
{
  tar -xvf etcd-v3.4.15-linux-amd64.tar.gz
  sudo mv etcd-v3.4.15-linux-amd64/etcd* /usr/local/bin/
}
```

Distribute generated server-to-client and peer-to-peer certificates

```bash
{
  sudo mkdir -p /etc/etcd1 /var/lib/etcd1
  sudo mkdir -p /etc/etcd2 /var/lib/etcd2
  sudo mkdir -p /etc/etcd3 /var/lib/etcd3

  sudo chmod 700 /var/lib/etcd1
  sudo chmod 700 /var/lib/etcd2
  sudo chmod 700 /var/lib/etcd3

  sudo cp etcd-ca-peer.pem \
    etcd-ca-cluster-to-client.pem \
    etcd-cluster-to-client-server.pem etcd-cluster-to-client-server-key.pem \
    etcd-peer-1.pem etcd-peer-1-key.pem \
    /etc/etcd1/

  sudo cp etcd-ca-peer.pem \
    etcd-ca-cluster-to-client.pem \
    etcd-cluster-to-client-server.pem etcd-cluster-to-client-server-key.pem \
    etcd-peer-2.pem etcd-peer-2-key.pem \
    /etc/etcd2/

  sudo cp etcd-ca-peer.pem \
    etcd-ca-cluster-to-client.pem \
    etcd-cluster-to-client-server.pem etcd-cluster-to-client-server-key.pem \
    etcd-peer-3.pem etcd-peer-3-key.pem \
    /etc/etcd3/
}
```

Create ETCD service units
```bash
{
cat <<EOF | sudo tee /etc/systemd/system/etcd1.service
[Unit]
Description=etcd1
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name etcd1 \\
  --cert-file=/etc/etcd1/etcd-cluster-to-client-server.pem \\
  --key-file=/etc/etcd1/etcd-cluster-to-client-server-key.pem \\
  --peer-cert-file=/etc/etcd1/etcd-peer-1.pem \\
  --peer-key-file=/etc/etcd1/etcd-peer-1-key.pem \\
  --trusted-ca-file=/etc/etcd1/etcd-ca-cluster-to-client.pem \\
  --peer-trusted-ca-file=/etc/etcd1/etcd-ca-peer.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://127.0.0.1:2370 \\
  --listen-peer-urls https://127.0.0.1:2370 \\
  --listen-client-urls https://127.0.0.1:2369 \\
  --advertise-client-urls https://127.0.0.1:2369 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster etcd1=https://127.0.0.1:2370,etcd2=https://127.0.0.1:2380,etcd3=https://127.0.0.1:2390 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd1
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

cat <<EOF | sudo tee /etc/systemd/system/etcd2.service
[Unit]
Description=etcd2
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name etcd2 \\
  --cert-file=/etc/etcd2/etcd-cluster-to-client-server.pem \\
  --key-file=/etc/etcd2/etcd-cluster-to-client-server-key.pem \\
  --peer-cert-file=/etc/etcd2/etcd-peer-2.pem \\
  --peer-key-file=/etc/etcd2/etcd-peer-2-key.pem \\
  --trusted-ca-file=/etc/etcd2/etcd-ca-cluster-to-client.pem \\
  --peer-trusted-ca-file=/etc/etcd2/etcd-ca-peer.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://127.0.0.1:2380 \\
  --listen-peer-urls https://127.0.0.1:2380 \\
  --listen-client-urls https://127.0.0.1:2379 \\
  --advertise-client-urls https://127.0.0.1:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster etcd1=https://127.0.0.1:2370,etcd2=https://127.0.0.1:2380,etcd3=https://127.0.0.1:2390 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

cat <<EOF | sudo tee /etc/systemd/system/etcd3.service
[Unit]
Description=etcd3
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name etcd3 \\
  --cert-file=/etc/etcd3/etcd-cluster-to-client-server.pem \\
  --key-file=/etc/etcd3/etcd-cluster-to-client-server-key.pem \\
  --peer-cert-file=/etc/etcd3/etcd-peer-3.pem \\
  --peer-key-file=/etc/etcd3/etcd-peer-3-key.pem \\
  --trusted-ca-file=/etc/etcd3/etcd-ca-cluster-to-client.pem \\
  --peer-trusted-ca-file=/etc/etcd3/etcd-ca-peer.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://127.0.0.1:2390 \\
  --listen-peer-urls https://127.0.0.1:2390 \\
  --listen-client-urls https://127.0.0.1:2389 \\
  --advertise-client-urls https://127.0.0.1:2389 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster etcd1=https://127.0.0.1:2370,etcd2=https://127.0.0.1:2380,etcd3=https://127.0.0.1:2390 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd3
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
}
```

> Important configuration options:
> --peer-(cert/key)-file - path to the file where cert/key file for communication with other ETCD peers will be used
> --peer-client-cert-auth - say ETCD to use peer certificate to authorize peer
> --peer-trusted-ca-file - the path to ca certificate file which will be used to validate peer certificates
>--(cert/key)-file  - path to the certificate/key file which will be used when communicate with clients
>--client-cert-auth - sat ETCD to use client certificate to authorize client
>-- trusted-ca-file - path to the ca certificate file which will be used to validate client certificates

And start etcd cluster
```bash
{
  systemctl enable etcd1 etcd2 etcd3
  systemctl start etcd1 etcd2 etcd3
}
```

## Verify

Get the list of etcd cluster members
```bash
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=etcd-ca-cluster-to-client.pem \
  --cert=etcd-cluster-to-client-client.pem \
  --key=etcd-cluster-to-client-client-key.pem
```

Result: 
```
3a57933972cb5131, started, controller-2, https://10.240.0.12:2380, https://10.240.0.12:2379, false
f98dc20bce6225a0, started, controller-0, https://10.240.0.10:2380, https://10.240.0.10:2379, false
ffed16798470cab5, started, controller-1, https://10.240.0.11:2380, https://10.240.0.11:2379, false
```

## Summary

In this section, we configured the ETCD cluster with 3 nodes. 
Each node has its certificate for communication with other nodes inside the ETCD cluster. To ensure that nodes use trusted certificates we generate certificate authority (CA certificate) and configure it on each node. 
To secure communication between the ETCD cluster and clients, we created several certificate pairs (client and server). This communication is also configured (now only on the server side) to trust only certificates signed with the usage of a separate CA. 
The CA for in-cluster communication and communication between ETCD cluster and client - different CAs. 

Next: [Configure API server](01-api-server.md   ) 