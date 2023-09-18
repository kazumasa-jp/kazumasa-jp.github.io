---
title: 低水準言語の技術
permalink: /docs/low-level-language-techniques-a/
tags: [MC6809, Z80, Assembly Language]
description: Some notes about Assembly Language
toc: true
---

## 数値を16進数へ変換する

### MC6809の場合
数値を16進数を表す文字へ変換するプログラムを考える。MC6809の場合は次のようになる。
```
    adda #$90
    daa
    adca #$40
    daa
```

### Z80の場合(1)
先のコードをZ80用に書き直すと、下記のようになる。

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

### Z80の場合(2)
Z80では次のような書き方もできる。
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

### C言語の場合

C言語の場合は、

```
    a = “0123456789ABCDEF”[a & 0x0f];
```

という有名な技法がある。

