---
title: Raspberry Pi 1をサーバーとして使う
permalink: /docs/install-vaultwarden-on-raspberrypi-e/
categories:
tags: [Raspberry Pi, docker, vaultwarden, nginx, samba, wsdd, unattended-upgrades, watchtower, rsync]
description:  
toc: true
---
## Vaultwardenとnginxのインストール

Vaultwardenとnginxを起動するためのdocker composeファイル(docker-compose.yml)を作成する。作業ディレクトリは~/work/dockerとする。

### docker-compose.ymlの作成

dockerには、composeというコマンドが用意してあり、これを使えば複数のコンテナ(サーバー)の管理ができる。

docker-compose.ymlの内容を下記に示す。

```
version: "3.9"
services:
  vaultwarden:
    image: vaultwarden/server:alpine
    restart: unless-stopped
    volumes:
      - /home/vw:/data/
    ports:
      - "81:80"
      - "3012:3012"

  nginx:
    image: nginx:alpine
    restart: always
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/vaultwarden.conf:ro
      - ../cert/:/etc/nginx/cert/:ro
    ports:
      - "80:80"
      - "443:443"
    command: [nginx-debug, "-g", "daemon off;"]
```
ここで、若干注意点がある。Raspberry Piが古いからなのか、OSにLite版を選んだからなのかわからないが、dockerのイメージにはlatestではなくalpineを使用する必要があった。

### nginx.confの作成

docker-compose.ymlが置かれているディレクトリ(~/work/docker/)にnginxの設定ファイルを作成する。nginx.confの内容は次の通り。

```
server {
    listen 80 default_server;
    server_name _;

    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl http2;
    ssl_certificate     /etc/nginx/cert/bitwarden.crt;
    ssl_certificate_key /etc/nginx/cert/bitwarden.key;

    # Allow large attachments
    client_max_body_size 128M;

    location / {
        proxy_pass http://vaultwarden:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /notifications/hub {
        proxy_pass http://vaultwarden:3012;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    location /notifications/hub/negotiate {
        proxy_pass http://vaultwarden:80;
    }
}
```
### vaultwardenとnginxの起動

"docker compose up"コマンドを使ってvaultwardenとnginxを起動する。

```
$ sudo docker compose up -d
```

設定ファイルが間違っているとリスタートを繰り返すことがあるため、docker psコマンドを使って、正しく動作していることを確認する

```
$ sudo docker ps

CONTAINER ID   IMAGE                       COMMAND                  CREATED       STATUS                 PORTS                                                                          NAMES
6cf1236669a8   nginx:alpine                "/docker-entrypoint.…"   x xxx ago   Up x xxx             0.0.0.0:80->80/tcp, :::80->80/tcp, 0.0.0.0:443->443/tcp, :::443->443/tcp       docker-nginx-1
65352cf24008   vaultwarden/server:alpine   "/usr/bin/entry.sh /…"   x xxx ago   Up x xxx (healthy)   0.0.0.0:3012->3012/tcp, :::3012->3012/tcp, 0.0.0.0:81->80/tcp, :::81->80/tcp   docker-vaultwarden-1
```
