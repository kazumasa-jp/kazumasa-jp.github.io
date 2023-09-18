---
title: Raspberry Pi 1をサーバーとして使う
permalink: /docs/install-vaultwarden-on-raspberrypi-a/
categories:
tags: [Raspberry Pi, docker, vaultwarden, nginx, samba, wsdd, unattended-upgrades, watchtower, rsync]
description:  
toc: true
---
## Raspberry Pi OS (Lite)のインストール

インストール自体はインストーラーの指示に従っていれば問題なく進められる。GUIは使用しないため、OSはLite版でよい。

```
OSを選ぶ > Raspberry Pi OS (other) > Raspberry Pi OS Lite (32-bit)
```

Raspberry Pi 1はUSBブートに対応していないため、microSDカードから起動する必要がある。Lite版であれば、容量は4GBでよい。

```
ストレージを選ぶ > SDHC Card - x.0 GB
```

再インストールを行う場合に備えて、インストーラーウインドウ右下にある歯車アイコンをクリックし、インストール時の設定を保存しておく。SSHを有効化し、公開鍵認証を選択した場合、authorized_keysの欄には公開鍵へのパスではなく、内容(ssh-rsaで始まる文字列)を貼り付ける。

```
ホスト名: servername.local
SSHを有効化する
　パスワード認証を使う
　公開鍵認証のみを許可する
　      ユーザーusernameのためのauthorized_keys： ssh-rsa xxxxxxxxx
ユーザー名とパスワードを設定する
　ユーザー名 username
　パスワード: password
Wi-Fiを設定する
　SSID:
　ステルスSSID
　パスワード
ロケールを設定する
　タイムゾーン: Asia/Tokyo
　キーボードレイアウト: jp
```
永続的な設定の欄で分かりにくいのは「テレメトリーを有効化する」というオプションだろう。これは、Raspberry Pi Imagerの使用状況やどのOSがよく使用されるかなどを調査するためのものだ([GitHub](https://github.com/raspberrypi/rpi-imager/blob/qml/README.md#telemetry))。
どちらを選んでもいい。

「保存」を押して詳細設定のウインドウを閉じたら、「書き込む」ボタンを押してSDカードへOSを書き込む。

SDカードをRaspberry Piに挿入して電源を入れ、Wi-FiやSSHが正しく動作することを確認する。以後はRaspberry Piでの作業になる。

## SSDのインストール

耐久性の問題から、microSDカードでの常時運用は不安があるため、ストレージにはSSDを使用する。前述の通り、Raspberry Pi 1はUSBブートに対応していないため、microSDカードでブートした後にSSDをマウントする必要がある。作業内容は次のページを参考にした([Raspberry Pi: Adding an SSD drive to the Pi-Desktop kit](https://www.zdnet.com/article/raspberry-pi-adding-an-ssd-drive-to-the-pi-desktop-kit/))。

まず、partedを使用してパーティションをシステム用とデータ用の二つに分割する。システム用には12GB、データ用には残りすべてを割り当てる。パーティションの名前は、

 - システム用：Linux System
 - データ用：Linux Home

 とし、フォーマットはLinuxで一般的なext4とした。partedのprintコマンドの出力は次の通り。今回は32GBのSSDを使用している。
```
Number  Start   End     Size    File system  Name          Flags
 1      1049kB  12.9GB  12.9GB  ext4         Linux System
 2      12.9GB  32.0GB  19.2GB  ext4         Linux Home
```

次にddを使用してルートファイルシステムをシステム用のSSDパーティションへコピーする。

```
dd if=/dev/mmcblk0p2 of=/dev/sda1 bs=4M iflag=fullblock oflag=direct status=progress
```

SSDをマウントしてhomeディレクトリのコピーとfstabの修正を行う。

```
## SSDをマウント
$ sudo mount /dev/sda1 /mnt
$ sudo mount /dev/sda2 /mnt/home

## パーミッションを保持したまま、ホームディレクトリをコピー
$ sudo cp -rp /home/* /mnt/home

## パーティションのPARTUUIDを調査する
$ sudo blkid
/dev/mmcblk0p1: LABEL_FATBOOT="bootfs" LABEL="bootfs" UUID="aaaa-aaaa" BLOCK_SIZE="512" TYPE="vfat" PARTUUID="mmmm-mmmm"
/dev/mmcblk0p2: LABEL="rootfs" UUID="bbbb-bbbb" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="nnnn-nnnn"
/dev/sda1: LABEL="rootfs" UUID="xxxx-xxxx" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="Linux System" PARTUUID="pppp-pppp"
/dev/sda2: UUID="yyyy-yyyy" BLOCK_SIZE="4096" TYPE="ext4" PARTLABEL="Linux Home" PARTUUID="qqqq-qqqq"

## fstabの内容を修正する
$ vi /mnt/etc/fstab
```

UUIDはfstabの内容は次の通り。

```
proc                    /proc   proc    defaults                0       0
PARTUUID=mmmm-mmmm      /boot   vfat    defaults                0       2
PARTUUID=pppp-pppp      /       ext4    defaults,noatime        0       1
PARTUUID=qqqq-qqqq      /home   ext4    defaults,noatime        0       3
```

/boot/cmdline.txtを修正し、PARTUUIDを使ってSSDをルートパーティションに指定する。内容は次の通り。修正する前にバックアップを取っておくほうがよいだろう。

```
console=serial0,115200 console=tty1 root=PARTUUID=pppp-pppp rootfstype=ext4 fsck.repair=yes rootwait
```

再起動して、SSDから起動すればOK。SSDからの起動に失敗したときは、cmdline.txtを元に戻せばSDカードから起動できる。

```
$ sudo reboot
...

## 再起動後、ファイルシステムが正しくマウントされているかを確認する。
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:16   0  29.8G  0 disk
├─sda1        8:17   0    12G  0 part /
└─sda2        8:18   0  17.8G  0 part /home
mmcblk0     179:0    0   7.4G  0 disk
├─mmcblk0p1 179:1    0   256M  0 part /boot
└─mmcblk0p2 179:2    0   7.2G  0 part
```
