---
title: バイナリ値をBCDに変換する(1)
author: Kazumasa
layout: post
tags: [Z80, Assembly Language, BCD]
description: Some notes about binary to BCD conversion
---
Z80で0〜99の数値をBCD表記(0x00〜0x99)へ変換する方法を考える。0〜99の数値を10で割った商と剰余を求める問題とも言える。

一番単純な方法は、被除数から10を何回引けるか数えるやり方だろう。
[MSX Resource Center](https://www.msx.org/forum/development/msx-development/bcdhex-conversion-asm)に記述例があった。
```
;*****************************
;     a(BIN) =>  a(BCD) 
;    [0..99] => [00h..99h]
;*****************************
bin2bcd:
	push bc
	ld   b,10
	ld   c,-1
div10:
    inc  c           ; 次の行でAからB(=10)を引く。引いた回数をCに保管
	sub  b           ;
	jr   nc,div10    ; A>=0の間は減算を続ける
	add  a,b         ; 引きすぎなので戻す。この時点でのCが商、Aが剰余
	ld   b,a
	ld   a,c         ; 商を上位ニブルに移動させるため、Cを16倍する
	add  a,a
	add  a,a
	add  a,a
	add  a,a
	or   b           ; Aレジスタの上位ニブルに商、下位ニブルに剰余が入る
	pop  bc
	ret
```
同じページにDAA命令を使用した方法も書いてあったので、こちらも引用しておく。

```
;*****************************
;     a(BIN) =>  a(BCD) 
;   [0..99] => [00h..99h]
;*****************************
bin2bcd:
	push bc
	ld   c,a   ; C:被除数
	ld   b,8   ; B:桁数
	xor  a     ; A=0とする
loop:
	sla  c
	adc  a,a   ; A=2*A+c
	daa        ; BCD表記へ変換
	djnz loop  ; 8回(Bレジスタの値)ループする
	pop  bc
	ret
```

被除数を上位ビットから見ていき、中間結果の最下位ビットに複写し、ビットシフトを行うことを繰り返している。

下記の式を計算しているのだが、各ステップ毎にDAA命令でBCD表記に直すことで、最終結果も正しいBCD表記が得られている。
```
((((((((((((c[7]*2)+c[6])*2)+c[5])*2)+c[4])*2)+c[3])*2)+c[2])*2)+c[1])*2+c[0]
=c[7]*2^7+c[6]*2^6+c[5]*2^5+c[4]*2^4+c[3]*2^3+c[2]*2^2+c[1]*2^1+c[0]
```
ネットを調べてみると、他にもいろいろ方法があったが、長くなるのでそれについては後日書くことにする。

