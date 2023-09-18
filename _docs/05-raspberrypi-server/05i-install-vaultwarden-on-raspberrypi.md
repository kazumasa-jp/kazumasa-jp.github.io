---
title: Raspberry Pi 1をサーバーとして使う
permalink: /docs/install-vaultwarden-on-raspberrypi-i/
categories:
tags: [Raspberry Pi, docker, vaultwarden, nginx, samba, wsdd, unattended-upgrades, watchtower, rsync]
description:  
toc: true
---
## Dockerコンテナの自動更新

unattended-upgradeではdockerイメージの更新はできないため、[watchtower](https://containrrr.dev/watchtower/)を使用する。

作業は~/work/watchtowerで行う。

### docker-compose.ymlの作成
次のようなdocker-compose.ymlファイルを作成する。[リンク先](https://www.jamescoyle.net/how-to/docker-compose-files/3323-docker-compose-file-for-watchtower)の設定をもとに、いくつかの行をコメントアウトしている。
watchtowerには、com.centurylinklabs.watchtower.enable = trueというラベルがつけられたコンテナのみをアップデートする機能があるが、今回は使用しないため、コメントアウトしている。リスタート中のコンテナもアップグレードの対象から外している。

```
version: '3'

services:
  watchtower:
    image: containrrr/watchtower
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
    environment:
      - WATCHTOWER_CLEANUP=true
#      - WATCHTOWER_LABEL_ENABLE=true
#      - WATCHTOWER_INCLUDE_RESTARTING=true
      - WATCHTOWER_SCHEDULE=0 0 4 * * 2
#    labels:
#      - "com.centurylinklabs.watchtower.enable=true"
```
WATCHTOWER_SCHEDULE変数で更新確認のタイミングを指示している。
6つの数字はそれぞれ、秒、分、時、日、月、曜日となっている。
上の設定で、火曜日の4:00に更新の確認を行う。

### Watchtowerの起動
Watchtowerの起動は、通常通り、docker composeで行う。

```
$ sudo docker compose up -d
```

正しく起動しているかどうか、docker psコマンドで確認する。
```
$ sudo docker ps
CONTAINER ID   IMAGE                COMMAND       CREATED   STATUS    PORTS     NAMES
3a95cdb23369  containrrr/watchtower "/watchtower" x xxx ago Up xx xxx 8080/tcp  watchtower-watchtower-1
...(以下省略)
```

設定時刻の確認は、docker logsコマンドで確認できる。docker psコマンドで確認したコンテナ名を用いて次のようにコマンドを入力する。

```
$ sudo docker logs watchtower-watchtower-1
time="YYYY-MM-DDTHH:MM:SS+09:00" level=info msg="Watchtower 1.5.3"
time="YYYY-MM-DDTHH:MM:SS+09:00" level=info msg="Using no notifications"
time="YYYY-MM-DDTHH:MM:SS+09:00" level=info msg="Checking all containers (except explicitly disabled with label)"
time="YYYY-MM-DDTHH:MM:SS+09:00" level=info msg="Scheduling first run: YYYY-MM-dd 04:00:00 +0900 JST"
time="YYYY-MM-DDTHH:MM:SS+09:00" level=info msg="Note that the first check will be performed in xx hours, xx minutes, xx seconds"
```
"Scheduling first run:"と書かれている行に次の更新確認時刻が表示される。
