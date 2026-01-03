---
layout: post
title: 通过Nginx设置你的git服务器
date: 2024-10-24 05:57 +0800
---

> 参考[git http-protocl](https://git-scm.com/docs/http-protocol)

> 参考Blog [Setting up a Git HTTP server with Nginx](https://esc.sh/blog/setting-up-a-git-http-server-with-nginx/)

## Target

1. 搭建一个轻量级的git http server
    - 用nginx 代理 git-http-backend 实现
    - 以某种方式将commit自动同步到其他公共gitserver.
2. git仓库缓存
    - 以某种方式请求server，能够自动拉取其他公共gitserver的仓库到本地。
    - 以某种方式更新这些仓库
3. 部署gitweb
4. 轻量级，不使用gitlab

## Http Protocol

这里简单解释一下git via Http的协议吧

使用git传输http包有两种协议：Dumb Http 和 Smart Http

### Dumb Http

Dumb Http 只能传输一些静态的文件，包括 info refs objects 等静态的文件。就像一个普通的http文件服务器一样。
所有数据的Content-Type建议为`text/plain;charset=utf-8`,不做强制要求(因为dumb clients不会检查Content-Type)，但一定不能是"application/x-git-"。


Dumb客户端会发起`/info/refs` 请求,服务器返回一些refs，以供下一次请求。

Dumb客户端:
```text
C: GET $GIT_URL/info/refs HTTP/1.0
```
服务器返回(Smart服务器支持dumb的行为):
```text
S: 200 OK
S:
S: 95dcfa3633004da0049d3d0fa03f80589cbcaf31	refs/heads/maint
S: d049f6c27a2244e12041955e262a404c7faba355	refs/heads/master
S: 2cb58b79488a98d2721cea644875a8dd0026b115	refs/tags/v1.0
S: a3c2e2402b99163d1d59756e5f207ae21cccba4c	refs/tags/v1.0^{}
```
### Smart Http

Smart 客户端支持一些更高级的操作。需要事先与服务器协商支持的命令。

Smart 客户端会发起`/info/refs?service=$servicename` 以询问服务器的能力。

目前 $servicename 只有两个: 
- git-upload-pack :服务器upload，客户端请求数据
- git-receive-pack :服务器receive，客户端上传数据

服务器会返回Content-Type: application/x-$servicename-advertisement
并在response body里告知自己的能力。

如果是Dumb服务器，则和上一节的内容相同。

Smart客户端：
```text
C: GET $GIT_URL/info/refs?service=git-upload-pack HTTP/1.0
```
Smart 服务器
```text
S: 200 OK
S: Content-Type: application/x-git-upload-pack-advertisement
S: Cache-Control: no-cache
S:
S: 001e# service=git-upload-pack\n
S: 0000
S: 004895dcfa3633004da0049d3d0fa03f80589cbcaf31 refs/heads/maint\0multi_ack\n
S: 003fd049f6c27a2244e12041955e262a404c7faba355 refs/heads/master\n
S: 003c2cb58b79488a98d2721cea644875a8dd0026b115 refs/tags/v1.0\n
S: 003fa3c2e2402b99163d1d59756e5f207ae21cccba4c refs/tags/v1.0^{}\n
S: 0000
```
之后 客户端会直接请求 $GIT_URL/$servicename 来和服务器传输数据

## 服务器部署

**注意:**

**各发行版的fastcgiwrap设置可能不一样。这里都是debian bookworm的默认配置**

### Smart Http server

简单概括一下需求：
1. 用git-http-backend做smart http服务
2. nginx 做网关转发请求
3. 部分dumb的文件请求直接由nginx处理

实现:
1. 服务请求 $GIT_URL=http(s)://domain/$git_repo
    - $git_repo 可能有多个分隔符 a/b/../z.git
2. http(s)://domain/git/.* 转发给git-http-backend处理
3. dumb服务请求$GIT_URL/info/refs
    - 这个不对应文件请求，backend处理
4. smart服务请求$GIT_URL/info/refs?service=$servicename
    - backend处理
5. dumb服务请求直接的文件
    - nginx处理
6. smart服务请求 $GIT_URL/git-upload-pack 与 $GIT_URL/git-receive-pack
    - backend 处理

根据[git-http-backend](https://git-scm.com/docs/git-http-backend#Documentation/git-http-backend.txt-AcceleratedstaticApache2x) 需要把这样的请求转发给git-http-backend:
```text
^/(
    .*/                             #匹配仓库名称repo
        (   HEAD |                  #repo/HEAD
            info/refs |             #repo/info/refs 服务发现请求
            objects/info/[^/]+ |    #objects/info 数据请求
            git-(upload|receive)-pack #smart http 的主要请求
        )
)$
```
这样的静态文件可以直接从文件系统获取:
```text
^/(.*/objects/[0-9a-f]{2}/[0-9a-f]{38})$
^/(.*/objects/pack/pack-[0-9a-f]{40}.(pack|idx))$
```

对应的location配置如下:
```text
# Accelerated static for git smart http
location ~ "^/(.*/objects/[0-9a-f]{2}/[0-9a-f]{38})$" {
    root /srv/git;
}
location ~ "^/(.*/objects/pack/pack-[0-9a-f]{40}.(pack|idx))$" {
    root /srv/git;
}
# For git smart http
location ~ ^(/.*/(HEAD|info/refs|objects/info/[^/]+|git-(upload|receive)-pack))$ {
    auth_basic off;
    fastcgi_pass unix:/var/run/fcgiwrap.socket;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME /usr/lib/git-core/git-http-backend;
    fastcgi_param GIT_PROJECT_ROOT /srv/git;
    fastcgi_param GIT_HTTP_EXPORT_ALL "";
    fastcgi_param PATH_INFO         $1;
}
```

**注意**：http不加用户验证(auth_basic off) http.receivepack 默认为false，加了用户验证之后默认为true.这将影响你上传数据.


### git cache

TODO

### Git web

另外我们还准备建立一个[gitweb](https://git-scm.com/book/en/v2/Git-on-the-Server-GitWeb)的服务，就像[linux kernel的git页面](https://git.kernel.org/)

Gitweb也比较简单，有一个现成的服务就叫gitweb，各大发行版应该都有这个包

- debian安装 gitweb
- archlinux安装 git 和 perl-cgi

记得修改一下/etc/gitweb.conf

上面smart http的配置已经考虑了location的兼容性问题，直接放在一个虚拟主机下是可行的。

```text
# for gitweb
location /index.cgi {
    root /usr/share/gitweb/;
    include fastcgi_params;
    gzip off;
    fastcgi_param SCRIPT_NAME $uri;
    fastcgi_param GITWEB_CONFIG /etc/gitweb.conf;
    fastcgi_pass unix:/var/run/fcgiwrap.socket;
}

location /{
    root /usr/share/gitweb/;
    index index.cgi;
}
```
