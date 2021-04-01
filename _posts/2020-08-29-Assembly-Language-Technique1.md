---
title: アセンブリ言語の技術(1)
author: Kazumasa
layout: post
tags: [Assembly Language]
description: Some notes about Assembly Language
---
DAA命令について調べていると、[daa命令の技](https://mechaag.tumblr.com/post/115002824017/daa%E5%91%BD%E4%BB%A4%E3%81%AE%E6%8A%80)というホームページがあったので、内容をメモしておく。

下記のプログラムは、数値を16進数を表す文字へ変換する。
```
    adda #$90
    daa
    adca #$40
    daa
```
上記ページの筆者は6809で学んだ人のようなので、Z80のコードではないが、有名な技法なのだろう。[stackoverflow](https://stackoverflow.com/questions/8119577/z80-daa-instruction)にZ80のコードがある。記述方法の違いだけで何も変わらない。
```
    and  15       ;
    add  a,90h    ; (1)
    daa           ; (2)
    adc  a,40h    ; (3)
    daa           ; (4)
```
(1)でAレジスタに0x90を足しているが、Aレジスタに入っている数字は0x00〜0x0fなので桁上がりは発生せず、キャリーフラグ、ハーフキャリーフラグは0である。

したがって、(2)のDAA命令の動作は、下表のいずれかになる。

| C Flag | HEX value in | H Flag | HEX value in | Number  | C flag |
| Before | upper digit  | Before | lower digit  | added   | After  |
| DAA    | (bit 7-4)    | DAA    | (bit 3-0)    | to byte | DAA    |
|--------|--------------|--------|--------------|---------|--------|
| 0      | 0-9          | 0      | 0-9          | 00      | 0      |
| 0      | 9-F          | 0      | A-F          | 66      | 1      |

ここまでの動作をまとめると、0xa〜0xfは(1)で0x9a〜0x9f、(2)で0x00〜05、(3)で0x41〜0x46、つまり'A'〜'F'に変換されている。一方、0x00〜0x09は、(1)で0x90〜0x99、(2)で0x90〜0x99、(3)で0xd0〜0xd9になっている。

いずれの場合もキャリーフラグ、ハーフキャリーフラグは立っていないので、最後の(4)のDAA命令の動作は、

| C Flag | HEX value in | H Flag | HEX value in | Number  | C flag |
| Before | upper digit  | Before | lower digit  | added   | After  |
| DAA    | (bit 7-4)    | DAA    | (bit 3-0)    | to byte | DAA    |
|--------|--------------|--------|--------------|---------|--------|
| 0      | 0-9          | 0      | 0-9          | 00      | 0      |
| 0      | A-F          | 0      | 0-9          | 60      | 1      |

となる。したがって、0x41〜0x46('A'〜'F')はそのままで、0xd0〜0xd9には0x60が足され、0x30〜0x39、つまり'0'〜'9'になる。

[daa命令の技](https://mechaag.tumblr.com/post/115002824017/daa%E5%91%BD%E4%BB%A4%E3%81%AE%E6%8A%80)には、Z80での別の書き方として次のコードが紹介してある。
```
    cp 0ah     ; (1)
    ccf
    adc a,30h  ; (2)
    daa        ; (3)
```

こちらはAレジスタを0x0a=10と比較して、(2)で0x00〜0x09なら0x30〜0x39、0x0a〜0x0fなら0x3b〜0x40になる。この場合のDAAの動作は、下表のようになる。

| C Flag | HEX value in | H Flag | HEX value in |  Number | C flag |
| Before |  upper digit | Before |  lower digit |   added |  After |
|    DAA |    (bit 7-4) |    DAA |    (bit 3-0) | to byte |    DAA |
|--------|--------------|--------|--------------|---------|--------|
|      0 |          0-9 |      0 |          0-9 |      00 |      0 |
|      0 |          0-8 |      0 |          A-F |      06 |      0 |
|      0 |          0-9 |      1 |          0-3 |      06 |      0 |

0x00〜0x09のときは2回目のDAA命令では何も起こらず、0x30〜0x39、つまり'0'〜'9'が最終結果となる。

0x0a〜0x0fのときは、0x0fの場合だけハーフキャリーフラグが立つが、DAA命令の演算内容としては変わらず、0x06が加算される。その結果、0x3b〜0x40は0x40〜0x46、つまり'A'〜'F'が得られる。

[daa命令の技](https://mechaag.tumblr.com/post/115002824017/daa%E5%91%BD%E4%BB%A4%E3%81%AE%E6%8A%80)には、C言語の例だが、
```
    a = “0123456789ABCDEF”[a & 0x0f];
```
というコードも紹介してある。これも有名な技法で、自分もどこかで見たことがある。

ついでなので、見たことがある技法の例としてもう一つ。[daa命令の技](https://mechaag.tumblr.com/post/115002824017/daa%E5%91%BD%E4%BB%A4%E3%81%AE%E6%8A%80)を書いた人の[ページ](https://mechaag.tumblr.com/post/122847195857/%E3%83%8A%E3%83%8E%E3%83%94%E3%82%B3%E6%95%99%E5%AE%A4)から。中間変数を使わずに２つの変数の内容を入れ替える方法だが、これも有名なコードで自分もどこかで見たことがある。
```
    a ^= b;
    b ^= a;
    a ^= b;
```

とはいえ、今はこういったひと目で動作がわからないようなコードは推奨されないだろう。また、6809ならともかく、Z80だとXORがAレジスタに対してしかできないから意味はなさそうだ。

さらに、同じく[daa命令の技](https://mechaag.tumblr.com/post/115002824017/daa%E5%91%BD%E4%BB%A4%E3%81%AE%E6%8A%80)を書いた人の[ページ](https://mechaag.tumblr.com/post/121974709877/8bit-cpu%E3%81%AE%E3%83%97%E3%83%AD%E3%82%B0%E3%83%A9%E3%83%9F%E3%83%B3%E3%82%B0%E3%83%86%E3%82%AF%E3%83%8B%E3%83%83%E3%82%AF2)から、XOR命令の応用例について。

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
