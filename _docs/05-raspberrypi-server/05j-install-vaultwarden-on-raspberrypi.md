---
title: Raspberry Pi 1をサーバーとして使う
permalink: /docs/install-vaultwarden-on-raspberrypi-j/
categories:
tags: [Raspberry Pi, docker, vaultwarden, nginx, samba, wsdd, unattended-upgrades, watchtower, rsync]
description:  
toc: true
---
## 定期バックアップ設定

データの定期バックアップには、rsyncとcronを使用する。前述の通り、バックアップ用に別のSSDを用意するが、バックアップ対象のデータ量が少ないため、ディスク領域を分割し、バックアップに使用しない領域には、CD-ROMのコピーなど、バックアップが不要な情報を格納している。

ディスクのマウント状況は次のようになっている。
```
$ lsblk

NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda           8:0    0 119.2G  0 disk
├─sda1        8:1    0   500M  0 part
├─sda2        8:2    0   128M  0 part
├─sda3        8:3    0    32G  0 part /mnt/backup
├─sda4        8:4    0     8G  0 part
└─sda5        8:5    0  78.6G  0 part /home/samba/username/Books
sdb           8:16   0  29.8G  0 disk
├─sdb1        8:17   0    12G  0 part /
└─sdb2        8:18   0  17.8G  0 part /home
mmcblk0     179:0    0   7.4G  0 disk
├─mmcblk0p1 179:1    0   256M  0 part /boot
└─mmcblk0p2 179:2    0   7.2G  0 part
```

設定は[リンク先1](https://qiita.com/hi-naoya/items/a120104538e679211185)や[リンク先2](https://linuxize.com/post/how-to-exclude-files-and-directories-with-rsync/)を参考にした。

### バックアップ用スクリプトの作成
~/bin/に次のようなスクリプトを作成し、実行属性をつけておく。

```
#!/bin/bash

dst_base=/mnt/backup/full/
src_base=/home/
EXCLUDE='--exclude lost+found/ --exclude samba/username/Books/ --exclude username/tmp/'
#TEST_OPT='--dry-run'

sudo rsync -av ${TEST_OPT} ${EXCLUDE} --delete ${src_base} ${dst_base}$(date "+%Y%m%d")
```

このスクリプトを実行すると、/homeディレクトリの内容が/mnt/backup/full/YYYYMMDD(日付)にフルバックアップが作成される。
3つのディレクトリ(lost+found, samba/username/Books, username/tmp)をバックアップ対象外とするため、--excludeオプションを3回指定している。

同様に差分バックアップのスクリプトを作成する。

```
#!/bin/bash

dst_base=/mnt/backup/diff/
src_base=/home/
prev_base=`\ls /mnt/backup/full/ | tail -n 1`
EXCLUDE='--exclude lost+found/ --exclude samba/username/Books/ --exclude username/tmp/'
#TEST_OPT='--dry-run'
sudo rsync -av ${TEST_OPT} ${EXCLUDE} --link-dest=../../full/${prev_base} ${src_base} ${dst_base}$(date "+%Y%m%d")
```

このスクリプトを使用すると、/mnt/backup/diff/YYYYMMDD(日付)に最後のフルバックアップからの差分バックアップが作成される。
バックアップを実行するrsyncコマンドでは、--link-destオプションを使用して、基準となるバックアップを指定している。
最新のフルバックアップが含まれるディレクトリはprev_base=...で取得し、--link-destには差分バックアップ保存先からの相対パスを指定する。

### バックアップのスケジュール設定
crontab -eコマンドを使用して、バックアップスケジュールを設定する。

```
$ sudo crontab -e
```
設定内容は次の通り。
```
# m h  dom mon dow   command
00 04 16   *   * /home/username/bin/backup-full.sh
00 02 *    *   5 /home/username/bin/backup-diff.sh
```
行頭の5桁の数字は、スクリプトを実行する時刻設定となっており、分、時、日、月、曜日の順に並んでいる。それほど頻繁に使用するサーバーではないため、毎月16日の4:00にフルバックアップを行い、毎週金曜日の2:00に差分バックアップを行うスケジュールとしている。