---
title: 低水準言語の技術
permalink: /docs/low-level-language-techniques-i/
tags: [C言語]
description: Some notes about Assembly Language
toc: true
---

## 変数の入れ替え

XOR演算の応用例として、中間変数を使わずに２つの変数の内容を入れ替える方法がある。

```
    a ^= b;
    b ^= a;
    a ^= b;
```

これも有名な手法だが、今はこういったひと目で動作がわからないようなコードは推奨されないだろう。
また、6809ならともかく、Z80だとXORがAレジスタに対してしかできないから意味はなさそうだ。
