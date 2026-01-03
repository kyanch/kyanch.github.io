---
layout: post
title: LUKS basic
date: 2024-10-03 18:12 +0800
---


## LUKS your disk

```bash
# format as Luks with default configuration
$ cryptsetup LuksFormat <device> 
# open Luks as /dev/mapper/name
$ cryptsetup open <device> <name>
```
All your operation to disk is mapped to `/dev/mapper/name`

## Configure your system

1. edit `/etc/mkinicpio.conf` 
  - add luks support to make your os able you decrypt your disk
  - add `systemd` `sd-vconsole` `sd-encrypt`
```bash
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)
```

2. edit `/etc/crypttab.initramfs`
  - the format is: name, underlying device, passphrase,cryptsetup options
  - for example: `root UUID=xxx none fido-device=auto`

3. remember rebuild your ramfs, `sudo mkinicpio -P`

## add fido2 key

1. install package `libfido2`
2. `systemd-cryptenroll /dev/disk`
  - check keyslots of your disk
3. `systemd-cryptenroll /dev/disk --fido2-device=auto --fido2-with-client-pin=0`
  - enroll your fido2 key

That's it.
