---
title: Raspberry Pi 1をサーバーとして使う
permalink: /docs/install-vaultwarden-on-raspberrypi-g/
categories:
tags: [Raspberry Pi, docker, vaultwarden, nginx, samba, wsdd, unattended-upgrades, watchtower, rsync]
description:  
toc: true
---
## Raspberry Piの常時運用

Raspberry Piを常時稼働させておくために、ソフトウェアの自動更新、自動バックアップの設定を行う。
同一ディスク内にバックアップを行った場合、ディスク破損時にバックアップデータも読みだせなくなる可能性があるため、バックアップ用には別のSSDを使用する。
