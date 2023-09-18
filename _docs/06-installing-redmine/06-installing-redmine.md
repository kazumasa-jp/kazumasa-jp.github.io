---
title: Arch LinuxへのRedmineのインストール
permalink: /docs/installing-redmine/
tags: [Arch Linux, redmine, nginx, mariadb, php]
description: I installed redmine in my Arch Linux machine. Install process is not so straight forward. 
toc: true
---
## コメント

いろいろと調べるために自宅のPCにRedmineをインストールしてみることにした。

インストール先のパソコンは、Dell Inspiron 11 3000。安価なPCで、性能も低いので、今はArch Linuxを動かしている。Arch Linuxはドキュメントがしっかりしているので、Redmineも簡単にインストールできるだろうと思っていたが、Webアプリケーションについての自分の知識不足もあって、一苦労したのでメモを残しておく。

バイナリパッケージによるインストールは手軽な半面、何を設定してあるかがわかりにくく、余計な苦労をしてしまうことがある。今回はまさにそのケースで、定評の有るArch Wikiを見ながら作業を進めても、だいぶ手間取った。redmineインストールディレクトリのオーナーを最初にhttpに変更し、rubyは2.6を使うようにしていれば、スムーズに進んでいたのだが…