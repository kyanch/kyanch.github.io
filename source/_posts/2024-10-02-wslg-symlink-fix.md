---
layout: post
title: Arch WSLg Symlink Fix
date: 2024-10-02 09:10 +0800
---

**FIRST OF ALL: YOU MUST SATISFY THE SYSTEM REQUIREMENT OF WSLG.**

The wslg is almost out of box. Except the latest GPU driver, there is no configuration required.

But the Arch WSL2 has some problems and the soultion is here: [WSL Symlink Fix](https://github.com/microsoft/wslg/issues/1032#issuecomment-2310369848)

## For Wayland Symlink Fix

All magic happens here:
```bash
/bin/sh -c 'USER_ID=$(id -u); find /mnt/wslg/runtime-dir -name "wayland-*" -type s -exec ln -sf {} /run/user/$USER_ID/ \;'
```

And Here is completed solution:
```bash

mkdir -p "${HOME}/.config/systemd/user"
cat <<"EOF" > "${HOME}/.config/systemd/user/wsl-wayland-symlink.service"
[Unit]
Description=Create Wayland symlinks for WSL
After=default.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'USER_ID=$(id -u); find /mnt/wslg/runtime-dir -name "wayland-*" -type s -exec ln -sf {} /run/user/$USER_ID/ \;'

[Install]
WantedBy=default.target
EOF
systemctl --user daemon-reload
systemctl --user enable wsl-wayland-symlink.service
systemctl --user start wsl-wayland-symlink.service

```
