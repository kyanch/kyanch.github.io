---
layout: post
title: gpg smartcards tips
date: 2024-11-03 20:37 +0800
---


## 使用Pcscd连接到smartcards

记得先启用pcscd
```
sudo systemctl enable pcscd.service --now
```

`~/.gnupg/scdaemon.conf`
```
pcsc-driver /usr/lib/libpcsclite.so
card-timeout 5
disable-ccid
```
你可能需要关闭gpg的scdaemon
```
gpgconf --kill scdaemon
```

## 管理多个smartcards

安装工具 openpgp-card-tools，依赖pcsclite和ccid

[ArchWiki - OpenPGP-card-tools](https://wiki.archlinux.org/title/OpenPGP-card-tools)
