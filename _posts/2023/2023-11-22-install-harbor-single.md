---
layout:       post
title:        "Harbor单实例安装"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Harbor
    - CloudNative
---

[Harbor](https://goharbor.io/)是一个开源成熟的Registry项目。很适合用来搭建私有化容器镜像仓库。下面列出单节点harbor的搭建步骤。

本教程中的域名和TLS证书都是自签的，因此涉及到认证和DNS的配置，在正式环境中，域名和证书如果已经提前准备好了，可以跳过一些自签和配置安全访问的步骤，请根据实际情况操作。

## 安装Docker

安装Docker的方法在不同操作系统是不一样的，请参考：[Install Docker Engine](https://docs.docker.com/engine/install/)。

下面以常见的Ubuntu为例：

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install docker and docker-compose
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-compose
```

需要注意的是，除了安装docker本身之外，harbor还会依赖`docker-compose`，请注意自行安装。

## 准备TLS证书

我们需要证书来配置harbor的HTTPS访问。如果你手头没有证书，可以使用openssl自签一个。注意自签的证书只适合调试和内部使用。

另外，需要一个域名来访问harbor，如果你没有域名，也可以随便指定一个，但是随便指定的域名需要修改`/etc/hosts`才能访问。

假设域名为`myhub.service.cn`，下面开始生成自签证书：

```bash
# Create a dir, to store all your cert files
mkdir /path/to/harbor-certs
cd /path/to/harbor-certs

# Generate a CA certificate private key.
openssl genrsa -out ca.key 4096

# Generate the CA certificate.
openssl req -x509 -new -nodes -sha512 -days 3650 \
 -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=myhub.service.cn" \
 -key ca.key \
 -out ca.crt

# Generate a private key.
openssl genrsa -out myhub.service.cn.key 4096

# Generate a certificate signing request (CSR).
openssl req -sha512 -new \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=example/OU=Personal/CN=myhub.service.cn" \
    -key myhub.service.cn.key \
    -out myhub.service.cn.csr

# Generate an x509 v3 extension file.
cat > v3.ext <<-EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1=myhub.service.cn
EOF
# Use the v3.ext file to generate a certificate for your Harbor host.
openssl x509 -req -sha512 -days 3650 \
    -extfile v3.ext \
    -CA ca.crt -CAkey ca.key -CAcreateserial \
    -in myhub.service.cn.csr \
    -out myhub.service.cn.crt

# Convert crt to cert, for use by Docker.
openssl x509 -inform PEM -in myhub.service.cn.crt -out myhub.service.cn.cert
```

注意将上面的`myhub.service.cn`换成你自己的域名。

复制Docker需要的证书文件：

```bash
sudo mkdir -p /etc/docker/certs.d/myhub.service.cn

sudo cp /path/to/harbor-certs/{myhub.service.cn.crt,myhub.service.cn.key,ca.crt} /etc/docker/certs.d/myhub.service.cn

sudo systemctl restart docker
```

## 安装Harbor

下面，准备开始安装Harbor，你可以按照需要到[release](https://github.com/goharbor/harbor/releases)下载在线或离线安装包。安装包是tar.gz解压格式的。

下载之后，解压安装包并进入：

```bash
tar -xzvf harbor-xxx-installer-version.tgz
cd harbor
```

编辑harbor的配置文件：

```bash
mv harbor.yml.tmpl harbor.yml
vim harbor.yml
```

关于配置文件的具体描述，请参考官方文档，这里列出几个必要的配置项：

```yaml
# 这是刚刚配置的域名，我们将使用这个域名来访问harbor
hostname: myhub.service.cn

http:
  port: 80

https:
  port: 443
  # 这是刚刚生成的证书文件路径，请根据实际情况填写
  certificate: /path/to/harbor-certs/myhub.service.cn.crt
  private_key: /path/to/harbor-certs/myhub.service.cn.key

database:
  # 初始Harbor数据库密码
  password: xxxx

# 初始admin账户密码
harbor_admin_password: xxxx

# Harbor储存数据的路径
data_volume: /path/to/harbor-data
```

请根据实际情况进行配置，完成之后，就可以开始安装Harbor了：

```bash
sudo ./install.sh

# 查看Harbor组件的状态
sudo docker-compose ps
```

## Docker客户端配置

**如果你的域名和证书是生产环境的，这里可以跳过**

由于我们的域名是假的，并且证书是自签的，因此在docker客户端还需要进行一些配置才能从Harbor拉取镜像。

首先，要配置假域名的IP解析，为Harbor服务器的IP：

```bash
sudo echo "\nhub.service.cn <harbor-ip>\n" >> /etc/hosts
```

执行前请替换上面命令的域名和IP。

从harbor服务器下载证书文件，加到本地Docker的信任证书中，以确保TLS验证可以通过（你也可以配置跳过TLS验证，但是不建议）：

```bash
sudo mkdir -p /etc/docker/certs.d/myhub.service.cn

sudo scp <user>@<ip>:/path/to/harbor-certs/{myhub.service.cn.crt,myhub.service.cn.key,ca.crt} /etc/docker/certs.d/myhub.service.cn

sudo systemctl restart docker
```

执行前请将`<user>`和`<ip>`替换为harbor服务器的用户名和IP地址。

这样，Docker就能正常访问harbor，可以尝试用admin账号登录一下：

```bash
docker login myhub.service.cn -u admin -p 'xxxx'
```

## 浏览器配置

**如果你的域名和证书是生产环境的，这里可以跳过**

harbor控制台需要在浏览器打开，一般我们直接访问443端口即可。但是上面的证书是自签的，浏览器因为安全配置可能无法打开不受信任的域名，因此可能需要将自签的证书加到浏览器配置中，才能直接访问harbor控制台。

我们先将证书文件下载到本地：

```bash
scp <user>@<ip>:/path/to/harbor-certs/ca.crt ca.crt
```

不同浏览器导入证书的方法不同，下面以119版本的Chrome为例。

打开浏览器设置，找到安全配置：

![chrome-config-security](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2023-11-22-install-harbor-single/chrome-config-security.png)

在安全配置中找到证书管理：

![chrome-config-certs](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2023-11-22-install-harbor-single/chrome-config-certs.png)

在`Authorities`标签页，导入上面下载的`ca.crt`文件：

![chrome-config-cert-import](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2023-11-22-install-harbor-single/chrome-config-cert-import.png)

导入的时候，一定要勾选信任所有类型：

![chrome-config-cert-trust](https://raw.githubusercontent.com/fioncat/fioncat.github.io.images/main/2023-11-22-install-harbor-single/chrome-config-cert-trust.png)

这样，你就可以在浏览器中直接通过域名来访问harbor控制台了。
