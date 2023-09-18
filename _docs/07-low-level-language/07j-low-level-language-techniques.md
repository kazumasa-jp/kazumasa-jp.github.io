---
title: 低水準言語の技術
permalink: /docs/low-level-language-techniques-j/
tags: [Assembly Language, x80, C言語]
description: Some notes about Assembly Language
toc: true
---

# 最小値を求める

XOR命令を使用して次のコードと等価な動作を実現する。

```
    unsigned a;
    unsigned b;
    a = (a < b) ? a : b;
```

上記のコンパイル結果をアセンブリ言語(x86)で書くと次のようになる。

```
    cmp eax,ebx       ; (1) 大小比較を行い、キャリーフラグを変化させる
    sbc edx,edx       ; (2) edxの値は何でも良い。
                      ;     キャリーフラグの値に応じて、
                      ;     edxは0x0000_0000、
                      ;     または0xffff_ffffになる。
    xor eax,ebx       ; (3)
    and eax,edx       ; (4)
    xor eax,ebx       ; (5)
```

肝は(4)の部分だ。edx=0x0000_0000のときは、(3)の結果如何にかかわらず、(4)の論理積を計算するところでeax=0x0000_0000となる。(5)では0とebxの排他的論理和を取るので、eaxはebxになる。

一方、edx=0xffff_ffffのときは、(4)の論理積ではeaxの値は変わらない。結局、(3)〜(5)ではebxとの排他的論理和を2回とっていることになるので、最終結果はeaxに等しくなる。
