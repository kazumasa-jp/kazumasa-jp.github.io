---
title: アセンブリ言語の技術(2)
author: Kazumasa
layout: post
tags: [Assembly Language]
description: Some notes about Assembly Language
---
サイクルスチールについて調べていたら、2ちゃんねるの議論の中に8ビットマシンでの乗算の話が有るのを見つけた。

6809であれば乗算命令があるので、1命令11クロックで済むが、Z80だとどう書くかという話だ。数パターンのコードが書いてあったが、やっていることは基本的に同じだろう。一番最後に出ていたシンプルなコードを載せておく。
```
; 8bitの掛け算(H * E -> HL)
    LD D,0
    LD L,D
    LD B,8
MULT1:
    ADD HL,HL     ; HLを2倍(左シフト)
    JR NC,MULT2   ; シフト前のHLの最上位ビットが0なら、MULT2へ飛び、MULT1を繰り返す
    ADD HL,DE     ; シフト前のHLの最上位ビットが1なら、Eを足す
MULT2:
    DJNZ MULT1    ; MULT1を繰り返す(最大8回:レジスタのビット数)
    RET
```

Hレジスタを上位ビットから見ていき、ビットが立っていれば中間結果(HLレジスタ)を2倍、立っていなければ2倍+Eレジスタを計算している。この考え方で乗算結果が得られることは筆算で確かめられる。

調べてみると、これと同じコードは[MSX Assembly Page](http://map.grauw.nl/articles/mult_div_shifts.php)にも見られる。おそらく、これもよく使われる方法なのだろう。
