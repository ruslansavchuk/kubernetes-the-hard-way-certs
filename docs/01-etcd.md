# Конфігурація ETCD

Стан всього кластера кубернетес зберігається у базі данних. В переважній більшості "дистрибутивів" в якості такої використовується [etcd](https://github.com/etcd-io/etcd). Саме тому при конфігурації кластера все починається із etcd.

## Трохи при нашу інсталяцію

Etcd є розподіленою ключ-значенняю системою зберігання даних, яка використовується для керування конфігурацією, координації та інших задач у розподілених системах. Розроблена компанією CoreOS.

Звісно дуже просто можна "підняти" базу данних etcd з 1-ю нодою в докері (приклад можна знайти наприклад [тут](https://etcd.io/docs/v2.3/docker_guide/)). Але для того щоб хоч трохи розібратись із "тонкостями" роботи, ми підемо трохи іншим шляхом.

В рамках даної лабораторної роботи ми розгорнемо кластер etcd з 3 нод. Вся комунікація у відконфігурованому кластері буде відбуватись тільки по зашифрованих каналах.

## Схема комунікації у кластері

Для початку нам потрібно розібратись хто і з ким збирається комунікавути, і що нам потрібно для того щоб зробити комунікацію безпечною (мається на увазі які сертифікати нам потрібні для того щоб досягнути таких цілей).

Як видно із картинки вище, є кілька принципово відмінних каналів комунікації:

1. Комунікація мід нодами кластеру
2. Комунікація між клієнтом і кластером

На щастя в etcd є широкі можливості для того щоб зробити вся комунікацію безпечною (більш детільно [тут](https://etcd.io/docs/v3.4/op-guide/security/)). Якщо коротко, то вся комунікація в кластері буде відбуватись із використанням TLS сертифікатів які ми згенеруємо самі. При при комунікації, буде також перевірятись і те чи правильним [ca](https://en.wikipedia.org/wiki/Certificate_authority) сертифікатом підписаний сертифікат який використовується.

### Комунікація між нодами кластеру

Як можна бачити з картинки вище, вся комунцікая мід нодами нашого кластеру буде відбуватись через порти 2380 спеціально відконфігуровані для цього на кожній ноді etcd (тут власне ніякоє оригінальності). Більш цікавим (і відмінним від того що  я бачив ранцше) є те, що ми будемо генерувати окремі сертифікати для комунікації між нодами, прицьому вони будуть підписані своїм власним ca сертифікатом.

### Комунікація між клієнтом та кластером

Для комунікації з клієнтами кластеру, ми будемо використовувати порт 2379. І знову ж таки, для того щоб зробити комунікацію безпечною ми згенеруємо окремі сертифікати підписані власним ca сертифікатом (в даному випадку ми будемо використовувати окремі сертифікати як для клаєнта так і для кластеру).


## Генерація сертифікатів

на даному етапі варто приєднатись до manage ноди так як ми почнемо уже генерувати необхідні нам сертифікати і розповсяджувати їх на всі інші машини.

Це можна зробити виконавши команду
```bash
{
  wget -q --show-progress --https-only --timestamping \
    https://github.com/cloudflare/cfssl/releases/download/v1.4.1/cfssl_1.4.1_linux_amd64 \
    https://github.com/cloudflare/cfssl/releases/download/v1.4.1/cfssljson_1.4.1_linux_amd64

  mv cfssl_1.4.1_linux_amd64 cfssl
  mv cfssljson_1.4.1_linux_amd64 cfssljson
  chmod +x cfssl cfssljson
  sudo mv cfssl cfssljson /usr/local/bin/
}  
```

### Сертифікати для комунікації між нодами кластеру

Почати звісно потрібно із того що ми згенеруємо ca сертифікат з використанням якого ми будемо генерувати всі наступні сертифікати потрібні для комунікації між нодами

```bash
{
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

Результат:
```
etcd-ca-peer-key.pem
etcd-ca-peer.csr
etcd-ca-peer.pem
```

Тепер ми згереруємо сертифікати для комунікації між нодами кластеру

```bash
{
cat > etcd-peer-csr.json <<EOF
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
  etcd-peer-csr.json | cfssljson -bare etcd-peer
}
```

Результат:
```
etcd-peer-key.pem
etcd-peer.csr
etcd-peer.pem
```

### Сертифікати для комунікації між клієнтом і кластером

Як і на попередньому кроці, почнемо ми звісно із ca сертифікату

```bash
{
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

Результат:
```
etcd-ca-cluster-to-client-key.pem
etcd-ca-cluster-to-client.csr
etcd-ca-cluster-to-client.pem
```

Тепер згенеруємо сертифікати як буде використовувати сервер для комунікації з клієнтами
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

Результат:
```
etcd-cluster-to-client-server-key.pem
etcd-cluster-to-client-server.csr
etcd-cluster-to-client-server.pem
```

А тепер клієнтський сертифікат для з'єднання із сервером
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

Результат:
```
etcd-cluster-to-client-client-key.pem
etcd-cluster-to-client-client.csr
etcd-cluster-to-client-client.pem
```

## Налаштування кластеру etcd

Тепер прийшов час відконфігурувати всі ноди кластеру etcd.
> так як всі пожальші команди потрібно буде виконати на кожній ноді, рекомендується використати для цього tmux (який уже встановлений у manage контейнері на якому ми генерували всі команди) який дає можливість виконувати команди паралельно у кількох відкритих табах (деталі можна знайти [тут](http://man.openbsd.org/OpenBSD-current/man1/tmux.1#synchronize-panes))
> :setw synchronize-panes on

Завантажимо etcd
```
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.15/etcd-v3.4.15-linux-amd64.tar.gz"
```

Розпакувати і помістити etcd у диреторію /usr/local/bin/
```
{
  tar -xvf etcd-v3.4.15-linux-amd64.tar.gz
  sudo mv etcd-v3.4.15-linux-amd64/etcd* /usr/local/bin/
}
```

Скопіюємо раніше згенеровані (і перенесені на ноду) сертифікати у директорію etcd
```
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
    etcd-cluster-to-client-client.pem etcd-cluster-to-client-client-key.pem \
    etcd-peer.pem etcd-peer-key.pem \
    /etc/etcd1/

  sudo cp etcd-ca-peer.pem \
    etcd-ca-cluster-to-client.pem \
    etcd-cluster-to-client-server.pem etcd-cluster-to-client-server-key.pem \
    etcd-cluster-to-client-client.pem etcd-cluster-to-client-client-key.pem \
    etcd-peer.pem etcd-peer-key.pem \
    /etc/etcd2/

  sudo cp etcd-ca-peer.pem \
    etcd-ca-cluster-to-client.pem \
    etcd-cluster-to-client-server.pem etcd-cluster-to-client-server-key.pem \
    etcd-cluster-to-client-client.pem etcd-cluster-to-client-client-key.pem \
    etcd-peer.pem etcd-peer-key.pem \
    /etc/etcd3/
}
```

Тепер нам потрібно отримати айпі адресу ноди
```
apt-get install dnsutils -y
INTERNAL_IP=$(dig +short $ETCD_NAME | head -n 1)
```

Створити `etcd1.service` юніт файл для нашого etcd кластеру (їх має бути 3 штуки)
```
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
  --peer-cert-file=/etc/etcd1/etcd-peer.pem \\
  --peer-key-file=/etc/etcd1/etcd-peer-key.pem \\
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
```

Створити `etcd2.service` юніт файл для нашого etcd кластеру (їх має бути 3 штуки)
```
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
  --peer-cert-file=/etc/etcd2/etcd-peer.pem \\
  --peer-key-file=/etc/etcd2/etcd-peer-key.pem \\
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
```

Створити `etcd3.service` юніт файл для нашого etcd кластеру (їх має бути 3 штуки)
```
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
  --peer-cert-file=/etc/etcd3/etcd-peer.pem \\
  --peer-key-file=/etc/etcd3/etcd-peer-key.pem \\
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
```

Тепер нам потрібно заставити його "піднятись"
```
{
  systemctl enable etcd1
  systemctl start etcd1
  systemctl enable etcd2
  systemctl start etcd2
  systemctl enable etcd3
  systemctl start etcd3
}
```

## Перевірка

Отримаємо спосок "мембурів" etcd кластеру
```
sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=etcd-ca-cluster-to-client.pem \
  --cert=etcd-cluster-to-client-client.pem \
  --key=etcd-cluster-to-client-client-key.pem
```

Результат: 
```
3a57933972cb5131, started, controller-2, https://10.240.0.12:2380, https://10.240.0.12:2379, false
f98dc20bce6225a0, started, controller-0, https://10.240.0.10:2380, https://10.240.0.10:2379, false
ffed16798470cab5, started, controller-1, https://10.240.0.11:2380, https://10.240.0.11:2379, false
```

Далі: [Конфігуруємо апі сервер](04-kube-apiserver.md) 
