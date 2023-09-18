---
title: Raspberry Pi 1をサーバーとして使う
permalink: /docs/install-vaultwarden-on-raspberrypi-c/
categories:
tags: [Raspberry Pi, docker, vaultwarden, nginx, samba, wsdd, unattended-upgrades, watchtower, rsync]
description:  
toc: true
---
## オレオレ証明書の作成

HTTPS接続を有効にするために、いわゆるオレオレ証明書を作成する([参考: Create your own Certificate Authority (CA) using OpenSSL](https://arminreiter.com/2022/01/create-your-own-certificate-authority-ca-using-openssl/))

### 作業準備
ホームディレクトリの下に作業用のworkディレクトリを作成する。その下にcertディレクトリを作成し、certディレクトリの下で作業を行う。

### 認証局(Certificate Authority:CA)用の秘密鍵の作成

コマンド"openssl genpkey"を使い、秘密鍵(private-ca.key)を作成する。
```
$ openssl genpkey -algorithm RSA -aes128 -out private-ca.key -outform PEM -pkeyopt rsa_keygen_bits:2048
```
鍵生成アルゴリズムにはRSAを使用し、暗号方式にはAES-128を用いている。出力フォーマットはPEM、生成される鍵の長さは2048ビットを指定している。

### 認証局の証明書を作成する
生成した秘密鍵(private-ca.key)を用いて、認証局の証明書(self-signed-ca-cert.crt)を作る。
```
$ openssl req -x509 -new -nodes -sha256 -days 3650 -key private-ca.key -out self-signed-ca-cert.crt
```
コマンド"openssl req"は、証明書署名要求(certificate request)の処理を行うコマンドだが、オプション"-x509"を使用すると自己署名証明書(self signed certificate)の作成ができる。

新規の証明書を作成するためにオプション"-new"を指定し、SHA-256を使用してメッセージダイジェストを作成する("-sha256")。有効期間は3650日とした("-days 3650")。

オプション"-nodes"は、opensslのマニュアルを見ると秘密鍵を作成する際に暗号化しないという指示になるようだが、今回は秘密鍵を生成しないため必要ないような気もする。

### Vaultwardenサーバー用の秘密鍵を作成する
認証局用の秘密鍵の生成と同じ方法で、サーバー用の秘密鍵(bitwarden.key)を作成する。
```
$ openssl genpkey -algorithm RSA -out bitwarden.key -outform PEM -pkeyopt rsa_keygen_bits:2048
```
### Vaultwardenサーバー用の証明書署名要求（Certificate Signing Request:CSR）ファイルを作成する
再びコマンド"openssl req"を使用して、今度はVaultwardenサーバー用の証明書署名要求ファイル(bitwarden.csr)を作成する。秘密鍵には先ほど作成したbitwarden.keyを使用する。
```
$ openssl req -new -key bitwarden.key -out bitwarden.csr
```
自己署名証明書を作成するわけではないので、オプション"-x509"や"-sha256"は使用しない。

### vaultwardenサーバー用の証明書を作成する
証明書署名要求に追加するX.509拡張をbitwarden.extとして保存する。
```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = servername.local
DNS.2 = www.servername.local
```
ブロック"[alt_names]"の"DNS.1", "DNS.2"は各自の環境に合わせて書き換える。

### vaultwardenサーバー証明書に署名する
コマンド"openssl x509"で、証明書署名要求(bitwarden.csr)にX.509拡張(bitwarden.ext)を追加し、自己署名認証局(self-signed-ca-cert.crt, private-ca.key)で署名を行う。
```
openssl x509 -req -in bitwarden.csr -CA self-signed-ca-cert.crt -CAkey private-ca.key -CAcreateserial -out bitwarden.crt -days 365 -sha256 -extfile bitwarden.ext
```
オプション"-CAcreateserial"で証明書のシリアル番号を自動的に生成している。

### 認証局の証明書(self-signed-ca-cert.crt)をWindowsへインストールする
何らかの方法(ftpやUSBメモリなど)で認証局の証明書をWindows PCへコピーし、「信頼されたルート証明機関」としてインストールする。

