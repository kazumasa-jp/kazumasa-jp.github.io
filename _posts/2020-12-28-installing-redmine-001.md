---
title: Arch LinuxへのRedmineのインストール (1)
author: Kazumasa
layout: post
tags: [Arch Linux, redmine, nginx, mariadb, php]
description: I installed redmine in my Arch Linux machine. Install process is not so straight forward. 
---
# はじめに

会社でRedmineを使うことになったが、メンバーがあまりこの手のツールに詳しくない上、自分に管理者権限がないので、いろいろと調べるために自宅のPCにRedmineをインストールしてみることにした。

インストール先のパソコンは、Dell Inspiron 11 3000。安価なPCで、性能も低いので、今はArch Linuxを動かしている。Arch Linuxはドキュメントがしっかりしているので、Redmineも簡単にインストールできるだろうと思っていたが、Webアプリケーションについての自分の知識不足もあって、一苦労したのでメモを残しておく。

# 必要なパッケージのインストール

redmineの他にデータベース、Webサーバー、アプリケーションサーバーが必要になる。すべてpacmanを使用してArch Linux公式リポジトリからインストールする。

1. redmine
   - pacmanでredmineをインストールすると、同時にruby2.6もインストールされる。ruby2.6のインストール先は/opt/ruby2.6になる。
   ```
   $ sudo pacman -S redmine install
   ```
   redmine関連の設定は後述する。
     
2. データベース
   - データベースにはMariaDBを使用する。redmineの実行には直接関係はないが、管理用にPHPMyAdminもインストールする。
   ```
   $ sudo pacman -S mariadb
   $ sudo pacman -S phpmyadmin
   ```
     
3. Webサーバー&アプリケーションサーバー
   - Webサーバーにはnginxを使用する。一昔前はWebサーバーといえばApacheだったような気がするが、最近はnginxが人気らしい。今回は使用するPCが非力なので、高速、軽量という評判通りであれば助かる。nginxでPHPを使うためにphp-fpmもインストールする。
   - アプリケーションサーバーにはPhusion Passengerを使用する。そもそもアプリケーションサーバーとはなんぞやというところがわかっていないのだが、Ruby on Railsを使用するアプリケーションで、Passengerはよく使われているようだ。
   ```
   $ sudo pacman -S nginx-mod-passenger
   $ sudo pacman -S php-fpm
   ```
