---
title: Raspberry Pi 1をサーバーとして使う
permalink: /docs/install-vaultwarden-on-raspberrypi-h/
categories:
tags: [Raspberry Pi, docker, vaultwarden, nginx, samba, wsdd, unattended-upgrades, watchtower, rsync]
description:  
toc: true
---
## aptパッケージの自動更新

aptパッケージの自動更新については、[リンク先](https://www.mikan-tech.net/entry/raspi-unattended-upgrades)の内容を参考にした。

まず、unattended-upgradesをインストールする。

```
$ sudo apt update
$ sudo apt install unattended-upgrades
```

unattended-upgradesの設定ファイルは、/etc/apt/apt.conf.d/50unattended-upgradesになる。
アップグレード対象ファイルは、Unattended-Upgrade::Origins-Patternで指定する。

全パッケージをアップグレードの対象にする場合、
```50unattended-upgradesの一部
Unattended-Upgrade::Origins-Pattern {
        "o=*,a=*";
};
```
のようにしておけばよいとのことだが、今回は、apt-cacheの出力を使って設定する。
```
$ sudo apt-cache policy
 100 /var/lib/dpkg/status
     release a=now
 500 https://pkg.ltec.ch/public bullseye/main armhf Packages
     release o=LTEC AG,n=bullseye,l=LTEC AG public,c=main,b=armhf
     origin pkg.ltec.ch
     ...(以下省略)
```
releaseで始まる行のo=, a=, n=, l=, c=を使用する。
```50unattended-upgradesの一部
Unattended-Upgrade::Origins-Pattern {
        // Raspberry Pi
        "o=Raspberry Pi Foundation,a=stable,n=${distro_codename},l=Raspberry Pi Foundation,c=main";
        "o=Raspbian,a=stable,n=${distro_codename},l=Raspbian,c=rpi";
        "o=Raspbian,a=stable,n=${distro_codename},l=Raspbian,c=non-free";
        "o=Raspbian,a=stable,n=${distro_codename},l=Raspbian,c=contrib";
        "o=Raspbian,a=stable,n=${distro_codename},l=Raspbian,c=main";
        // Docker
        "o=Docker,a=${distro_codename},l=Docker CE,c=stable";
        // WSDD
        "o=LTEC AG,n=bullseye,l=LTEC AG public,c=main";
};
```
n(codename)には${distro_codename}というマクロを使用している。/etc/debian_versionの内容に応じて、buster, bullseyeなど、OSバージョンのコードネームが設定される。

自動再起動の設定は、50unattended-upgradesで次のような設定を行う。

``` 50unattended-upgradesの一部
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
```

また、不要になったパッケージを自動削除するために次の設定を行っておく。

```
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
```

最後にアップデート時刻の設定を行う。アップデートはパッケージのダウンロードとパッケージのインストールの二つのステップで行われ、それぞれに時刻の設定が必要である。


ダウンロード時刻の設定は次のコマンドで行う。
```
$ sudo systemctl edit apt-daily.timer
```
エディタが開いたら、次のように入力する。これで月曜日の0:00から2:00の間にダウンロードが行われるようになる。
```
[Timer]
OnCalendar=
OnCalendar=Mon 0:00
RandomizedDelaySec=2h
Persistent=true
```

同様に、アップデート時刻の設定は次のコマンドで行う。
```
$ sudo systemctl edit apt-daily-upgrade.timer
```
次の設定で、月曜日の3:00から4:00の間にアップグレードが行われるようになる。
```
[Timer]
OnCalendar=
OnCalendar=Mon 3:00
RandomizedDelaySec=60m
Persistent=true
```

設定を反映させるために、apt-daily.timer, apt-daily-upgrade.timerを再起動する。

```
$ sudo systemctl restart apt-daily.timer
$ sudo systemctl restart apt-daily-upgrade.timer
```
