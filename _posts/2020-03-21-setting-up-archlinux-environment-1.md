---
title: Setting up Arch Linux environment (1)
author: Kazumasa
layout: post
tags: [Arch Linux, trouble, update, i3wm]
---
pacman -SyuでArch Linuxを更新したところ、起動しなくなってしまった。原因はカーネルを更新したときに/dev/sda1を/bootにマウントしていなかったことで、起動時のOSのバージョン(/dev/sda1に含まれている)が古いままなので、新しいカーネルモジュールを読み込めないということのようだ。

それはともかく、ウインドウマネージャ周りの設定はおおよそ終わったと思う。
やったことは
- 特殊キーに音量やバックライトの調整を割り当て
- ステータスバーに音量を表示
- EmacsとChromiumにホットキーを割り当て
- Rofiのインストール
- タッチパッドをオフにするスクリプトの作成
- マウスを使わずにログアウトするスクリプトの作成
ぐらいだが、普段使うぶんには問題はない。問題はこれまで使っていたVirtual Box上のLinux Mintが使いにくく感じてしまうことぐらいだろう。

あとはEmacsを含む開発環境。こちらは時間がかかりそうなので、ぼちぼち進めることにしよう。
