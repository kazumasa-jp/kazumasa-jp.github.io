---
title: Arch LinuxへのRedmineのインストール
permalink: /docs/installing-redmine-c/
tags: [Arch Linux, redmine, nginx, mariadb, php]
description: I installed redmine in my Arch Linux machine. Install process is not so straight forward. 
toc: true
---
## データベース(MariaDB)の準備
1. [MariaDB(ArchWiki)](https://wiki.archlinux.jp/index.php/MariaDB)に従って、データベースの初期化、MariaDBサービスの起動を行ない、セキュリティ向上スクリプトを実行する。
```
$ sudo mariadb-install-db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
$ sudo systemctl start mariadb
$ sudo systemctl enable mariadb
$ sudo mariadb-secure-installation
```
2. 次にredmineユーザーを作成する。
```
$ sudo mysql -u root -p
...
MariaDB [(none)]> CREATE DATABASE redmine CHARACTER SET UTF8;
MariaDB [(none)]> CREATE USER 'redmine'@'localhost' IDENTIFIED BY 'my_password';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON redmine.* TO 'redmine'@'localhost';
```
3. tmpdirにはTMPFSを使うことにする。[MariaDB(ArchWiki)](https://wiki.archlinux.jp/index.php/MariaDB)の手順に従って作業を行う。
```
$ mkdir -pv /var/lib/mysqltmp
$ chown mysql:mysql /var/lib/mysqltmp
$ id mysql
uid=xxx(mysql) gid=yyy(mysql) groups=yyy(mysql)
```

4. /etc/fstabにtmpfsを追加する。
```
$ sudo vi /etc/fstab
```
```/etc/fstab
...
tmpfs   /var/lib/mysqltmp   tmpfs   rw,gid=yyy,uid=xxx,size=100M,mode=0750,noatime   0 0
```
5. MariaDBの設定ファイルでtmpdirを指定する。ついでにリモートアクセスの無効化と文字コードの設定も行う。
```
$ sudo vi /etc/my.cnf.d/server.cnf
```
```/etc/my.cnf.d/server.cnf
...
[mysqld]
skip-networking
collation_server=utf8mb4_unicode_ci
character_set_server=utf8mb4
tmpdir=/var/lib/mysqltmp
...
```
```
$ sudo vi /etc/my.cnf.d/client.cnf
```
```/etc/my.cnf.d/client.cnf
...
[client]
default-character-set=utf8mb4
...
```
```
$ sudo vi /etc/my.cnf.d/mysql-clients.cnf
```
```/etc/my.cnf.d/mysql-clients.cnf
...
[mysql]
default-character-set=utf8mb4
...
```
6. /usr/share/nginx/htmlにphpMyAdminへのシンボリックリンクを作っておく
```
$ sudo ln -s /usr/share/webapps/phpMyAdmin /usr/share/nginx/html/
```

ブラウザで http://localhost/phpMyAdmin/index.php にアクセスすると、PHPMyAdminが立ち上がることを確認しておく。
