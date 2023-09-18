---
title: Raspberry Pi 1をサーバーとして使う
permalink: /docs/install-vaultwarden-on-raspberrypi-d/
categories:
tags: [Raspberry Pi, docker, vaultwarden, nginx, samba, wsdd, unattended-upgrades, watchtower, rsync]
description:  
toc: true
---
## Dockerのインストール

Vaultwardenとnginxを動かすためにDockerをインストールする。Raspberry PiへのDockerのインストールは、Convinience scriptを使う方法がよく紹介されているが、[公式ドキュメント](https://docs.docker.com/engine/install/raspberry-pi-os/#install-using-the-repository)には"Only recommended for testing and development environments"と記載されているため、今回はaptを使ってインストールすることにした。

### Dockerリポジトリのセットアップ
Dockerは標準のリポジトリ設定でもインストールできるようだが、公式ドキュメントにしたがってリポジトリの設定を行う。
```
## インストールに必要なツールをインストールする前に、リポジトリを更新する
$ sudo apt-get update

## 必要なツールをインストールする
$ sudo apt-get install ca-certificates curl gnupg

## Dockerの公式GPG鍵をインストールする
$ sudo install -m 0755 -d /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/raspbian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
$ sudo chmod a+r /etc/apt/keyrings/docker.gpg

## DockerのAptリポジトリを登録する:
$ echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/raspbian \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

## リポジトリを更新する。
$ sudo apt-get update
```

### Dockerのインストール

aptを使ってDockerパッケージをインストールする。
```
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

インストールが正常に終了したことを確認するため、hello-worldを実行する。

```
$ sudo docker run hello-world
```