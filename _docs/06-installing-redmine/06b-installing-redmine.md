---
title: Arch LinuxへのRedmineのインストール
permalink: /docs/installing-redmine-b/
tags: [Arch Linux, redmine, nginx, mariadb, php]
description: I installed redmine in my Arch Linux machine. Install process is not so straight forward. 
toc: true
---
## Webサーバー(nginx)の起動
1. とりあえず、nginxとphp-fpmを起動しておく。設定が終わってから立ち上げてもいいだろう。
```
$ sudo systemctl start nginx
$ sudo systemctl enable nginx
$ sudo systemctl start php-fpm
$ sudo systemctl enable php-fpm
```
2. nginxの設定ファイルを編集する。load_module文で、Passengerモジュールを読み込む設定を行う必要がある。
```
$ sudo vi /etc/nginx/nginx.conf
```
```
load_module "/usr/lib/nginx/modules/ngx_http_passenger_module.so";
...
http {
    ...
    passenger_root /usr/lib/passenger;
    passenger_ruby /opt/ruby2.6/bin/ruby;
    passenger_logfile /var/log/nginx/passenger.log;
    ...
    server {
        ...
        location / {
            root /usr/share/nginx/html;
            index index.html
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
            root /usr/share/nginx/html;
        }

        location ~ \.php$ {
            root /usr/share/nginx/html;
            fastcgi_pass unix:/run/php-fpm/php-fpm.sock;
            fastcgi_index index.php;
            include fastcgi.conf;
        }

        location ~ ^/redmine(/.*|$) {
            alias /usr/share/webapps/redmine/public$1;
            passenger_base_uri /redmine;
            passenger_app_root /usr/share/webapps/redmine;
            passenger_document_root /usr/share/webapps/redmine/public;
            passenger_enabled on;
            rails_env production;
        }
    }
}
```
3. nginxを再起動する。
```
% sudo systemctl restart nginx
```

4. 続いてPHPの設定を行う。MariaDBにアクセスするためにMySQL拡張モジュールを読み込む。また、タイムゾーンを東京に指定しておく。
```
$ sudo vi /etc/php/php.ini
```
```
[PHP]
...
open_basedir = /usr/share/webapps/:/usr/share/nginx/html/:/tmp/
...
extension=mysqli
...
extension=pdo_mysql
...
[Date]
...
date.timezone = Asia/Tokyo
...
```

5. 必要はなさそうだが、動作確認のためにphp-fpmのログを取るように設定する。
```
$ sudo vi /etc/php/php-fpm.d/www.conf
```
```
[www]
...
prefix = /var/log/php-fpm/$pool
...
access.log = access.log
...
```
6. php-fpmを再起動する。
```
$ sudo systemctl restart php-fpm
```
