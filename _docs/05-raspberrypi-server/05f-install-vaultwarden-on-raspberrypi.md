---
title: Raspberry Pi 1をサーバーとして使う
permalink: /docs/install-vaultwarden-on-raspberrypi-f/
categories:
tags: [Raspberry Pi, docker, vaultwarden, nginx, samba, wsdd, unattended-upgrades, watchtower, rsync]
description:  
toc: true
---
## Sambaのインストール

Vaultwardenのデータだけでは、SSDの容量が余るので、ファイルサーバーとしても運用することにした。
[参考サイト](https://pc.watch.impress.co.jp/docs/column/ubuntu/1501933.html)を参考にし、Sambaを立ち上げる。

Sambaは古くから使われているソフトウェアで、Unix系のOS上でWindowsネットワークの機能を実現する。dockerイメージも存在するが、公式のものではないため、アップデートの頻度などに不安が残る。Sambaについてはaptによるインストールを行う。

### ユーザーと共有フォルダの作成

最初にsamba用のユーザーとグループを作成する。
```
$ sudo adduser --system --group --no-create-home samba
```
sambaをログインできないユーザーとするために、--systemオプションを指定している。また、ユーザー作成と同時に--groupオプションでユーザー名と同じ名前のグループ(samba)を作成する。

--no-create-homeはホームディレクトリ(/home/samba/)を作成しないことを指示するオプションだが、後の作業で"/home/samba/"内に共有ディレクトリを作成するため、--no-create-homeは指定しなくてもよい。

次に共有ディレクトリを作成する。今回は"/home/samba/public/"をゲストアクセス可能な共有ディレクトリとする。

```
$ sudo mkdir -p /home/samba/public
$ sudo chown samba: /home/samba/public
$ sudo chmod 775 /home/samba/public
```

必要に応じて、個人専用の共有フォルダを作成する。

```
$ sudo mkdir /home/samba/username
$ sudo chown username: /home/samba/username
$ sudo chmod 775 /home/samba/username
```

Sambaのユーザー登録を行い、パスワードを設定する。

```
$ sudo pdbedit -a samba
$ sudo pdbedit -a username
```

### 設定ファイルの作成

Sambaの設定ファイルは/etc/samba/smb.confである。
デフォルトのsmb.confには多くの説明コメントが記載されている。必要に応じて設定を行えばよいが、今回は下記リストの追加、または修正の箇所を変更した。
```
## testparmは設定確認用のコマンド
$ testparm

## Global parameters
[global]
        dos charset = CP932                     # 追加、または修正
        guest account = samba                   # 追加、または修正
        log file = /var/log/samba/log.%m
        logging = file
        map to guest = Bad User
        max log size = 1000
        obey pam restrictions = Yes
        pam password change = Yes
        panic action = /usr/share/samba/panic-action %d
        passwd chat = *Enter\snew\s*\spassword:* %n\n *Retype\snew\s*\spassword:* %n\n *password\supdated\ssuccessfully* .
        passwd program = /usr/bin/passwd %u
        server role = standalone server
        unix password sync = Yes
        usershare allow guests = Yes
        idmap config * : backend = tdb

[homes]
        browseable = No
        comment = Home Directories
        create mask = 0700
        directory mask = 0700
        valid users = %S

[printers]
        browseable = No
        comment = All Printers
        create mask = 0700
        path = /var/spool/samba
        printable = Yes

[print$]
        comment = Printer Drivers
        path = /var/lib/samba/printers

[Public]                                # 追加
        create mask = 0664              # 追加
        directory mask = 0775           # 追加
        force group = samba             # 追加
        force user = samba              # 追加
        guest ok = Yes                  # 追加
        guest only = Yes                # 追加
        path = /home/samba/public       # 追加
        read only = No                  # 追加

[Username]                              # 追加
        create mask = 0664              # 追加
        directory mask = 0775           # 追加
        force group = username          # 追加
        force user = username           # 追加
        path = /home/samba/username     # 追加
        read only = No                  # 追加
```
### Sambaの起動

aptを使ってSambaをインストールし、systemdへ登録する。
```
$ apt update
$ apt install samba
$ sudo systemctl restart smbd
```

### wsddのインストール

最近のWindowsでは、Sambaをインストールしただけでは、エクスプローラーのネットワーク欄にRaspberry Piが表示されない。この課題を解消するために[wsdd](https://github.com/christgau/wsdd)のインストールを行う。同様の機能を持つものにwsdd2があるが、aptを使用してインストールできるこちらを選択した。

Dockerのインストールと同様、リポジトリの登録から行う。公式ドキュメントには、公開鍵を/usr/share/keyrings/にインストールするように書かれているが、Dockerの公開鍵を/etc/apt/keyrings/ディレクトリに格納したので、wsddの公開鍵もこのディレクトリに保存する。
```
## 公開鍵をダウンロードし、バイナリ形式で保存する
wget -O- https://pkg.ltec.ch/public/conf/ltec-ag.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/wsdd.gpg
sudo chmod a+r /etc/apt/keyrings/wsdd.gpg
## リポジトリを登録する
source /etc/os-release
echo "deb [signed-by=/etc/apt/keyrings/wsdd.gpg] https://pkg.ltec.ch/public/ ${UBUNTU_CODENAME:-${VERSION_CODENAME:-UNKNOWN}} main" | sudo tee /etc/apt/sources.list.d/wsdd.list > /dev/null
## wsddのインストール
sudo apt update
sudo apt install wsdd
```

これでWindowsのエクスプローラー上にRaspberry Piが表示されるはずだ。
