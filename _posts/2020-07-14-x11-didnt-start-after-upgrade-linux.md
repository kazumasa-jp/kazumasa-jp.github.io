---
title: X11 didn't start after upgrading to linux-5.7
author: Kazumasa
layout: post
tags: [archlinux, update, trouble, X11, Xorg]
description: ArchLinuxのアップデート後にX11が立ち上がらない現象が発生した。xf86-video-intelをアンインストールして解決。
---
前回の投稿からずいぶん間が空いてしまった。
メインマシンとして使っていたThinkPadが購入後2年も経たずに壊れてしまった。マザーボードが焦げていたので修理は諦め、ArchLinuxをインストールしたDell Inspiron 3000がメインマシンになってしまった（たぶん、もうLenovoは買わない）。だいぶ非力なマシンだが、メモリも増設したので、普段、Linuxで使うぶんにはそれほど不便は感じない。

ただ、ArchLinuxのアップデート後にX11が立ち上がらない現象が発生したので、忘れないようにメモを残しておく。正確には立ち上がっているのかもしれないが、LightDMが起動せず、画面が真っ暗になった。いろいろ苦労したが、xf86-video-intelをアンインストールすれば無事解決。
