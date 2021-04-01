---
title: Arch LinuxへのRedmineのインストール (5)
author: Kazumasa
layout: post
tags: [Arch Linux, redmine, nginx, mariadb, php]
description: I installed redmine in my Arch Linux machine. Install process is not so straight forward. 
---
# Easy Gantt Freeのインストール
最後にEasy Ganttをインストールする。

1. EasyGanttを[EasyRedmineのサイト](https://www.easyredmine.jp/redmine-gantt-plugin)からEasyGanttFree.zipを入手し、中に含まれているEasyGanttFree-4.x.zipを/usr/share/webapps/redmine/pluginsに展開しておく。

2. 次の手順でEasyGanttをインストールする。
```
$ cd /usr/share/webapps/redmine
$ sudo chown -R http:http plugins/easy_gantt
$ sudo -u http env PATH=/opt/ruby2.6/bin:$PATH bundle-2.6 install --without development test --path vendor/bundle/
$ sudo -u http env PATH=/opt/ruby2.6/bin:$PATH bundle-2.6 exec rake redmine:plugins NAME=easy_gantt RAILS_ENV=production
$ sudo -u http env PATH=/opt/ruby2.6/bin:$PATH RAILS_ENV=production bundle-2.6 exec rake db:migrate
$ sudo systemctl restart nginx
```

# 最後に
バイナリパッケージによるインストールは手軽な半面、何を設定してあるかがわかりにくく、余計な苦労をしてしまうことがある。今回はまさにそのケースで、定評の有るArch Wikiを見ながら作業を進めても、だいぶ手間取った。redmineインストールディレクトリのオーナーを最初にhttpに変更し、rubyは2.6を使うようにしていれば、スムーズに進んでいたのだが…
