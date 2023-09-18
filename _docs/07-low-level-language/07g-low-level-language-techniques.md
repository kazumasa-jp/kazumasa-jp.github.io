---
title: 低水準言語の技術
permalink: /docs/low-level-language-techniques-g/
tags: [Z80, Assembly Language, BCD]
description: Some notes about binary to BCD conversion
toc: true
---

## バイナリ値をBCDに変換する(5)

バイナリBCD変換(4)のアイデアをバイナリBCD変換(3)の方法へ応用する。

RRCAを4回行った際に、整数部に1が足しこまれた状態になるように補正してあるため、小数部の桁上がりを求めた後の処理が少し簡単になっている。基本的なアイデアはバイナリBCD変換(3)で説明してあるので、今日はソースコードにコメントを残すだけにしておく。

```
;;
;; 商を求める
;;
    LD A,B
    ADD A,32   ;; (1) 被除数を補正する
    ADD A,B    ;;     A=3/2*B+16
    ADD A,B    ;;
    RRA        ;; 
    LD C,A     ;; (2) Cレジスタに結果を保管する。Cレジスタのビットパターンをabcd_efghと書くことにする。
    RRCA       ;; (3) 4ビットの回転を行う(A=efgh_abcd)。これを4ビットのシフト操作(×1/16)とみなすと、
    RRCA       ;;     下位ニブルは整数部が入るが、上位のニブルには小数部が保持されている形になる。
    RRCA       ;;     下位のニブル(整数部)を計算してみると3/32+B+32/32≒0.09375*B+1となり、 
    RRCA       ;;     0.1*B+1の近似となる。ただし、誤差は6.25%残る。
    ADD A,C    ;; (4) ここまでの計算結果(A=3/32*B+32/32)を使って、もう少し整数部の精度を上げる。
               ;;     A*(1+1/16)=17/16*A
               ;;               =(17*3*B+17*32)/32/16
               ;;               =(51*B+512+32)/512
               ;;               =0.099609375*B+1+0.0625
               ;;     これで整数部の誤差は1%以下になる。
               ;;     Aの下位ニブルが整数部、上位ニブルが小数部となるので、この計算は、
               ;;        A   :      efgh_abcd
               ;;     +) A/16: efgh_abcd
               ;;     のようになる。
               ;;     知りたいのはefgh+abcdが桁上がりするかどうかである。
               ;;     そのためにはA+C=efgh_abcd+abcd_efghを計算すれば良い。
    CCF        ;; (5) abcdは(0.1*B+1)の近似になっている。efgh+abcdが桁上がりをしていなければ、
               ;;     abcdから-1を引かなければならない。
               ;;     ここでキャリーフラグを反転して、次のSBCでabcdを補正する
    SBC A,C    ;; 
    AND 0FH    ;; (6) A: 商
;;
;; 剰余を求める
;; バイナリBCD変換(3)と同じなので省略する。
;;
    LD H,A
    ADD A,A
    ADD A,A
    ADD A,H
    CPL
    RLCA
    ADC A,B
```