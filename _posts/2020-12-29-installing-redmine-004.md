---
title: Arch LinuxへのRedmineのインストール (4)
author: Kazumasa
layout: post
tags: [Arch Linux, redmine, nginx, mariadb, php]
description: I installed redmine in my Arch Linux machine. Install process is not so straight forward. 
---
# Redmineの設定
1. 作業を進める前に、/usr/share/webapps/redmineのオーナーを変更しておく。
```
$ cd /usr/share/webapps
$ sudo chown -R http:http redmine
```

2. データベースのアクセス設定を行う。database.ymlにはパスワードが書かれるため、ファイルのモードを変更する。
```
$ cd /usr/share/webapps/redmine
$ sudo cp config/database.yml{.example,}
$ sudo chmod 600 config/database.yml
$ sudo vi config/database.yml
```
```
...
production:
...
  username: redmine
  password: "my_password"
...
```

3. pacmanを使用してredmineをインストールした場合、いくつかの手順は必要なさそうだが、一応、書いておく。まず、必要なgemをインストールする。
```
$ sudo -u http env PATH=/opt/ruby2.6/bin:$PATH bundle-2.6 install --without development test --path=vendor/bundle
```

4. 何故か次のステップでlibpq.soが必要になるので、pacmanを使ってpostgresql-libsをインストールしておく。
```
$ sudo pacman -S postgresql-libs
```

5. 次にセッションストア秘密鍵を生成する。
```
$ sudo -u http env PATH=/opt/ruby2.6/bin:$PATH bundle-2.6 exec rake generate_secret_token
```

6. デフォルトデータでデータベースを作成する。
```
$ sudo -u http env PATH=/opt/ruby2.6/bin:$PATH RAILS_ENV=production bundle-2.6 exec rake db:migrate
$ sudo -u http env PATH=/opt/ruby2.6/bin:$PATH RAILS_ENV=production REDMINE_LANG=ja bundle-2.6 exec rake redmine:load_default_data
```

7. 手順1の段階で変わっていると思うが、念の為にディレクトリのパーミッションを設定する。
```
$ sudo chown -R http:http files log tmp public/plugin_assets
$ sudo chmod -R 755 files log tmp tmp/pdf public/plugin_assets
```

あとはlocalhost/redmineにアクセスすれば、redmineに接続できる。初期ユーザーはadmin/パスワード無しなので、すぐにパスワードを変更しておく。
