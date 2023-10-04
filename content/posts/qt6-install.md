---
title: "Qt6安装"
date: 2023-10-04T19:24:00+08:00
draft: true # Set 'false' to publish
description: ""
categories:
- code
tags:
- cpp
- qt
---

## 从国内源在线安装(无官方离线安装方式)

1. 下载在线安装器(USTC) - [qt-unified-linux-x64-online.run](http://mirrors.ustc.edu.cn/qtproject/official_releases/online_installers/qt-unified-linux-x64-online.run)
- [qt-unified-mac-x64-online.dmg](http://mirrors.ustc.edu.cn/qtproject/official_releases/online_installers/qt-unified-mac-x64-online.dmg)
- [qt-unified-windows-x64-online.exe](http://mirrors.ustc.edu.cn/qtproject/official_releases/online_installers/qt-unified-windows-x64-online.exe)

2. 启动时指定镜像源
```bash
*.exe --mirror https://mirror.nju.edu.cn/qt
*.exe --mirror https://mirrors.ustc.edu.cn/qtproject
```
3. 登录QT账号 
