---
layout:     post
title:      openssl创建TLS自签名CA证书
subtitle:   docker portainer.io endpoint tls
date:       2019-10-23
author:     李大毛
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - system
---

## 前言

openssl生成自签证书或CA证书的资料非常多，例如：

- [docs.azure.cn](https://docs.azure.cn/zh-cn/articles/azure-operations-guide/application-gateway/aog-application-gateway-howto-create-self-signed-cert-via-openssl)


本文描述了基于shell脚本执行，免于交互输入过程的，CA自签server/client双向证书认证的示例，
特别适用于自动化安装场景，例如docker的TSL安全设置。


### build-ca-cs-cert.sh <server-ip> <ca-password>


~~~
SERVER="$1"
PASSWORD="$2"

echo '创建临时目录'
mkdir build-ca-cs-cert
cd build-ca-cs-cert

echo '生成CA密钥'
openssl genrsa -aes256 -passout pass:$PASSWORD -out ca-key.pem 2048

echo '生成CA证书'
openssl req -passin "pass:$PASSWORD" -subj "/CN="$SERVER"/C=CN/ST=AH/L=HF/O=COM/OU=DEV" -new -x509 -days 3650 -key ca-key.pem -sha256 -out ca-cert.pem

echo '生成server端密钥'
openssl genrsa -out server-key.pem 2048

echo '生成server端CSR证书请求'
openssl req -subj "/CN=$SERVER" -new -key server-key.pem -out server.csr

echo '签发server端证书'
sh -c 'echo subjectAltName = IP:'$SERVER' >> extfile_server.cnf'
openssl x509 -req -days 3650 -in server.csr -CA ca-cert.pem -CAkey ca-key.pem -passin "pass:$PASSWORD" -CAcreateserial -out server-cert.pem -extfile extfile_server.cnf

echo '生成client端密钥'
openssl genrsa -out client-key.pem 2048

echo '生成client端CSR证书请求'
openssl req -subj '/CN=client' -new -key client-key.pem -out client.csr

echo '签发client端证书'
sh -c 'echo "extendedKeyUsage=clientAuth" >> extfile_client.cnf'
openssl x509 -req -days 3650 -in client.csr -CA ca-cert.pem -CAkey ca-key.pem -passin "pass:$PASSWORD" -CAcreateserial -out client-cert.pem -extfile extfile_client.cnf

echo '设置文件权限'
chmod 400 *-key.pem
chmod 444 *-cert.pem

echo 'CA证书'
openssl x509 -in ca-cert.pem -noout -text

echo 'server证书'
openssl x509 -in server-cert.pem -noout -text

echo 'client证书'
openssl x509 -in client-cert.pem -noout -text

~~~


### 用于docker的tsl鉴权

1. cp ca-cert.pem server-cert.pem server-key.pem /etc/docker/cert

2. vim /etc/docker/daemon.json

~~~
{
  ...
  "tlsverify": true,
  "tlscacert": "/etc/docker/cert/ca-cert.pem",
  "tlscert": "/etc/docker/cert/server-cert.pem",
  "tlskey": "/etc/docker/cert/server-key.pem"
}
~~~

3. systemctl restart docker


