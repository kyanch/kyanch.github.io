---
layout: post
title: thunderbird gpg
date: 2024-09-22 16:11 +0800
---

> [https://wiki.mozilla.org/Thunderbird:OpenPGP:Smartcards](https://wiki.mozilla.org/Thunderbird:OpenPGP:Smartcards)
## With PGP

Thunderbird 通过GPGme调用GPG功能。Windows下使用THunderbird需要安装GPG4Win，并且将bin_64加入PATH后方可正常使用

## With Security key

如需与安全密钥协同使用,需要配置external_gnupg：

thunderbird设置->常规->（最下方）配置编辑器-> mail.openpgp.allow_external_gnupg 改为 true


密钥配置、信任之类不再赘述，网络上很多，自行摸索也不难。
