---
layout:       post
title:        "在服务器外申请免费的Let's Encrypt证书"
subtitle:     "Apply a free certificate outside the server"
author:       "Fioncat"
header-style: text
catalog:      true
tags:
    - Linux
---

Let's Encrypt是一个免费的证书颁发机构，它让我们可以零成本搭建一个HTTPS站点。

要申请一个证书，我们需要一个域名。Let's Encrypt只要验证我们有这个域名的操作权限，就能为我们颁发证书。

> 碎碎念：Let's Encrypt只验证域名的操作权，严格意义不验证域名的所有权。因此你的域名如果被黑客黑了，他同样可以用这个域名申请证书。
>
> 因此Let's Encrypt的安全性是不如其他付费权威证书颁发机构的，某个站点即使有这种证书，也不能确保站点所有者是域名的所有者。
>
> 为了安全考虑，Let's Encrypt申请的证书有效期都比较短，无法申请到1年的长有效证书。
>
> 在申请和使用Let's Encrypt时一定要留意上述风险。并且要记得频繁刷新证书。使用免费的东西就要承担它的风险。

`Certbot`是一个常用的颁发Let's Encrypt证书的工具，它有很多种工作模式。其中`standalone`和`webroot`需要在服务器上面执行，并且服务器需要有外网入口，能将DNS解析记录配置到服务器上面。

但是很多情况下，可能存在以下情况：

- 需要通配符域名，例如`*.testsite.cn`，背后没有单一服务器。
- 服务器暂时没有暴露外网的能力。

这时候可以通过`manual`模式在外部申请证书，这就只要求：

- 申请证书的机器可以访问外网。
- 拥有对域名的操作权，可以增加TXT解析记录。

下面记录我为一个通配符域名申请证书的过程。

## 安装certbot

certbot是一个申请证书的工具，Linux安装方法如下：

```bash
sudo pacman -S certbot # archlinux
apt install certbot # Debain
yum install certbot # CentOS
```

## 手动申请证书

开始申请证书：

```bash
sudo certbot certonly --manual --preferred-challenges=dns-01
```

certonly会要求你输入邮箱、同意协议等操作，根据提示操作即可。

看到下面信息，表示certonly要求你输入你自己的域名，根据实际情况输入即可，可以输入多个域名。

```
Account registered.
Please enter the domain name(s) you would like on your certificate (comma and/or
space separated) (Enter 'c' to cancel): fioncat.online,*.fioncat.online
```

这里我需要为`*.fioncat.online`申请证书，但是为了验证`fioncat.online`域名是属于我自己的，还加上了主域名以用于验证。

## 验证域名所有权

当看到下面的信息时，表示可以开始验证域名所有权了：

```
Please deploy a DNS TXT record under the name:

_acme-challenge.fioncat.online.

with the following value:

Ak8z2EW17niuebu5It2CHlvnz1eyqnNEgW1BPv7KHkw
```

在你的域名提供商那里，加一条TXT类型的记录，主机为`_acme-challenge`，值为上面生成的随机字符串。

添加记录后，DNS不是立即生效的，这时候不要急忙在certbot中进入下一步，使用下面的命令确保DNS已经生效：

```console
$ dig -t txt _acme-challenge.fioncat.online

; <<>> DiG 9.18.25 <<>> -t txt _acme-challenge.fioncat.online
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 11716
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1410
;; QUESTION SECTION:
;_acme-challenge.fioncat.online.        IN      TXT

;; ANSWER SECTION:
_acme-challenge.fioncat.online. 600 IN  TXT     "Ak8z2EW17niuebu5It2CHlvnz1eyqnNEgW1BPv7KHkw"

;; Query time: 1776 msec
;; SERVER: 192.168.138.110#53(192.168.138.110) (UDP)
;; WHEN: Wed Apr 10 15:48:36 CST 2024
;; MSG SIZE  rcvd: 115
```

看到解析成功了，在certbot中回车进入下一步。

这时候，certbot会再输出一个随机的TXT让你加记录，重复上面的过程再加一遍就好了。注意第一条记录不要删除，你的域名应该有2条TXT记录。

使用`dig`命令确认两条记录都加好了之后，输入回车，certbot会开始验证并生成证书。如果一切正常，你会看到下面的信息：

```
Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/fioncat.online/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/fioncat.online/privkey.pem
This certificate expires on 2024-07-09.
These files will be updated when the certificate renews.
```

这已经列出了证书的位置，直接上传到你的服务器使用即可。

**注意证书是有有效期的，certbot会列出证书的过期时间。如果证书要过期了，手动重复上面的过程即可。**

你可以用下面命令手动检查证书过期时间：

```bash
sudo openssl x509 -in /etc/letsencrypt/live/fioncat.online/fullchain.pem -noout -dates
```

使用下面命令查看证书完整信息，包括域名等：

```bash
sudo openssl x509 -in /etc/letsencrypt/live/fioncat.online/fullchain.pem -noout -text
```

## 证书文件说明

在证书目录下，可以看到四个证书文件。分别为：

- `fullchain.pem`：证书文件，也叫`crt`文件。一般重命名为`<domain>.crt`。里面包含了完整的证书信息，服务端必须需要该文件。
- `privkey.pem`：私钥文件，也叫`key`文件。一般重命名为`<domain>.key`。服务端必须需要该文件。
- `chain.pem`：非必要，用于Nginx >= 1.3.7中的OCSP。
- `cert.pem`：不完整的证书文件，一般不使用。