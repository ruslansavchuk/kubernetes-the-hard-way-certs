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