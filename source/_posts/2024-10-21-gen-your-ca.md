---
layout: post
title: 生成你的CA证书
date: 2024-10-21 21:49 +0800
---

> 参考 [局域网内搭建浏览器可信任的SSL证书](https://www.tangyuecan.com/2021/12/17/%E5%B1%80%E5%9F%9F%E7%BD%91%E5%86%85%E6%90%AD%E5%BB%BA%E6%B5%8F%E8%A7%88%E5%99%A8%E5%8F%AF%E4%BF%A1%E4%BB%BB%E7%9A%84ssl%E8%AF%81%E4%B9%A6/)

> openssl使用方法参考[ArchWiki OpenSSL#Usage](https://wiki.archlinux.org/title/OpenSSL#Usage)

HTTPS使用ssl或tls来加密信息，所以我们这里使用openssl生成CA证书。

# 创建CA证书

第一步是创建一个密钥,这里用P-256曲线生成一个ECDSA密钥

```bash
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out myCA.key
```

第二步是通过密钥生成我们的自签名证书,也就是CA证书。CA证书默认过期时间是一个月，这里直接给100年

```bash
openssl req -key myCA.key -utf8 -x509 -new -days 36500 -out myCA.crt
```

这里你要输入机构信息, 如果你的输入中包含非ASCII字符，需要`-utf8`参数

| 参数名称                 | 缩写 | 参数值                                                                  |
| ------------------------ | ---- | ----------------------------------------------------------------------- |
| Country Name             | C    | 国家，CN中国,ZZ未知地区或国家                                           |
| State or Province Name   | ST   | 省名称                                                                  |
| Locality Name            | L    | 城市名称                                                                |
| Organization Name        | O    | 组织名称，个人使用就写姓名                                              |
| Organizational Unit Name | OU   | 组织单位名称，部门名称，没有就留空吧                                    |
| Common Name              | CN   | 常用名称，标识证书的名称，在系统里会以这个名称显示你的证书 My Custom CA |
| Email Address            |      | 邮件地址                                                                |

当然你也可以一行命令完成密钥创建和生成自签名证书：

```bash
openssl req -x509 -newkey EC -pkeyopt ec_paramgen_curve:P-256 -keyout myCA.key -out myCA.crt -day days
```

最后检查一下你的证书信息

```bash
openssl x509 -text -in myCA.crt
```

检查一下证书指纹

```bash
openssl x509 -noout -in myCA.crt -fingerprint -digest
# -digest 是 -md5, -sha1, -sha256, -sha512 中的一个
```

你现在已经有了自己的CA证书了，好好保管它。

你需要在系统里信任这个CA证书，这样以这个CA证书签名的服务器证书才会被信任。
如果你泄露了证书，那么别人也可以生成一个被信任的证书了。

# 为服务器生成证书签名请求

接下来，我们要为你的服务器创建证书了

第一步和创建CA证书的第一步相同，创建一个密钥。

```bash
openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out server.key
```

第二步，通过server.key这个密钥生成一个证书签名请求(CSR, Certificate Signing Request)，来让CA证书签名

```bash
openssl req -new -sha256 -config openssl.cnf -key server.key -out server.req
```

**注意**，这里也会让你输入机构信息，按照服务器所有者的机构信息进行填写。Common Name 建议填写QFDN

**注意**，你需要使用X.509证书的v3扩展中的subjectAltName字段来指定域名信息。现代系统会优先检查该选项。[RFC 2818 HTTP Over TLS](https://www.rfc-editor.org/rfc/rfc2818).

Common Name作为旧的指定域名的方法，也请指定域名，例如: \*.example.com

**注意**，生成CSR需要用到请求模板文件openssl.cnf，archlinux中自带的这个模板在/etc/ssl/openssl.cnf，你的系统里不出意外应该也有这个文件.

我们需要修改域名，请将这个文件复制一份，并修改其中`[v3_req]`的内容

```text
--- /etc/ssl/openssl.cnf        2024-09-04 00:32:59.000000000 +0800
+++ openssl.cnf 2024-10-21 23:06:36.552907206 +0800
@@ -231,12 +231,17 @@
 [ v3_req ]

 # Extensions to add to a certificate request

 basicConstraints = CA:FALSE
 keyUsage = nonRepudiation, digitalSignature, keyEncipherment
+subjectAltName = @alt_names
+
+[ alt_names ]
+DNS.1 = example.com
+DNS.2 = *.example.com

 [ v3_ca ]


 # Extensions for a typical CA
```

最后,你可能想确认一下生成的CSR的内容:

```bash
openssl req -noout -text -in server.req
```

CSR只用来发送给CA机构，让他们对证书签名生成crt文件,实际不会在服务器上使用。

所以，接下来，把CSR发给CA机构(你自己)让他们签名吧。

# CA机构为你签名服务器证书

你拿到了服务器给你的CSR，你需要通过自己的CA私钥和CA证书(其实就是带信息的公钥)处理CSR，生成一个服务器证书:

```bash
openssl x509 -req -extfile openssl.cnf -extensions v3_req -CA myCA.crt -CAkey myCA.key -in server.req -out server.crt -days 36500 -CAcreateserial -CAserial serial
```

你依然可以用同一个命令来查看证书信息

```bash
openssl x509 -text -in server.crt
```

CA签名有一点值得注意：签名会给证书一个唯一的序列号，这个序列号会被保存在一个位置，防止以后签名的证书序列号重复。这个位置定义在openssl.cnf里的

```text
[ ca-default ]
serial = XXXX
```

如果你有换机器签名的需求，建议迁移serial中的内容

## 安装你的CA证书

### ArchLinux

首先安装 `ca-certificates` 包,

然后将你的CA根证书放到 `/usr/share/ca-certificates/trust-source/anchors/` 目录下

运行`update-ca-trust`

具体的目录位置和命令名称不同发行版有所区别但流程基本一致

这里再附一个Debian的：

```bash
sudo apt-get install -y ca-certificates
sudo cp local-ca.crt /usr/local/share/ca-certificates
sudo update-ca-certificates
```

### Windows

一般来说，双击crt证书，可以打开Windows的证书导入向导。将CA证书存储在 `受信任的根证书颁发机构` 就行。

如果不可行，运行 `certmgr.msc`, 将证书导入到 `受信任的根证书颁发机构` 即可。

### Android

直接在系统设置里搜索`证书`，进入CA证书导入页面就行。

**注: 有些系统在导入CA证书后会提示`网络可能会受到监控`,如果你可以确保你的证书没有泄露,可以安全的忽略此提示。(尽管它会一直在任务栏提示) **

暂无MacOS设备，不讨论安装方法
