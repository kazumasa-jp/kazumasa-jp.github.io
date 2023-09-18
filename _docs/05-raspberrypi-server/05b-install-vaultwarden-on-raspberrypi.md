---
title: Raspberry Pi 1をサーバーとして使う
permalink: /docs/install-vaultwarden-on-raspberrypi-b/
categories:
tags: [Raspberry Pi, docker, vaultwarden, nginx, samba, wsdd, unattended-upgrades, watchtower, rsync]
description:  
toc: true
---
## サーバーの構成について

以上でRaspberry PiがSSDから起動するようになった。次にサーバーソフトウェアの設定を行う。

パスワードマネージャにはVaultwardenを使用する。VaultwardenはBitwardenというパスワードマネージャと互換性がある。VaultwardenもBitwardenもオープンソースソフトウェアだが、VaultwardenはBitwardenよりも軽量なので非力なRaspberry Pi 1で使用するならこちらだろう。もちろん、パソコンやスマートフォンで使用するクライアントソフトウェアはBitwardenのものが使用可能だ。

Vaultwardenは単独での運用も可能だが、HTTPS接続を可能とするためにnginxをリバースプロキシとして使用する。nginxも軽量なwebサーバーとして知られており、これもRaspberry Pi 1にとっては都合がよい。

また、nginxにはaptパッケージが用意されているが、vaultwardenには用意されていない。幸い、vaultwardenとnginxにはdockerという仮想化ソフトウェア用の公式イメージが公開されており、これらを使うとインストールが楽になるため、この二つはdocker上で動作させることにする。
