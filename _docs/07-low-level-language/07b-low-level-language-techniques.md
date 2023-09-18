---
title: 低水準言語の技術
permalink: /docs/low-level-language/low-level-language-techniques-b/
tags: [Z80, Assembly Language]
description: Some notes about Assembly Language
toc: true
---

# 8ビットCPUでの乗算

## Z80の場合

6809であれば乗算命令があるので1命令11クロックで済むが、Z80の場合は次のように書けばよい。

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
