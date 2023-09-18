---
title: "Tiny Basicを読む"
permalink: /docs/comments-on-tinybasic/
toc: true
---
## はじめに

Tiny Basicはその名の通り小さなBasic処理系で、いくつかのバージョンが存在するが、[Palo Alto版Tiny Basic](https://en.wikipedia.org/wiki/Li-Chen_Wang#Palo_Alto_Tiny_BASIC)が最も有名なもののようだ(Wikipediaのリンクから入手できるものは、いくつかの命令を追加したSHERRY BROTHERS TINY BASIC VERSION 3.1である)。

Palo Alto版Tiny BasicのターゲットCPUはインテル8080、プログラムサイズは2KBである。その名の通り小さなプログラムであるがアセンブリ言語で書かれているため、ソースコードはそれなりに長い。本稿ではこのTinyBasicの解析を行う。

文章に残すにあたって用語の定義を決めておく。時々使用方法を間違えるかもしれないが、気づいたら適宜追加、修正する。


コマンド
  - キーボード等から入力され、Tiny Basic処理系に指示を与えるもの

ステートメント/文
  - Basicプログラム中の文

キーワード
  - "FOR"、"IF"など、コマンドやステートメントに含まれる予約語

プログラムテキスト
  - Tiny Basicのプログラム

プログラムテキスト領域
  - Tiny Basicのプログラムが保存されるメモリ領域

入力バッファ
  - プログラム外から入力された文字列が一時保管されるメモリ領域

機械語プログラム
  - Tiny Basicインタープリタ自身を構成する機械語プログラム

FOR変数
  - FORステートメントの処理に使用される5つの変数(LOPPT, LOPLN, LOPLMT, LOPINC, LOPVAR)

コマンドテーブル
  - キーワードとキーワードに対応する機械語プログラムの開始位置を要素として持つテーブル。
  - TAB1, TAB2, TAB4, TAB5, TAB6, TAB8があり、EXECルーチンで使用される。

ルーチン
  - ある処理を行うプログラムの塊を表す。この文書では特に記述がない限り機械語プログラムの一部を指す。

## 初期化

CP/Mのアプリケーションプログラム領域は0100H番地から始まる。

```
       ORG  100H      ;OF CPM.
START  JMP  NINIT      ;GO TO INITIALIZATION ROUTINE.	JIF
```

Tiny Basicインタープリタが開始されると、まずNINIT(0AA0H番地)へ飛ぶ。

```
       ORG	0AA0H
NINIT: LXI	H,RST1		;POINT TO BEGINNING OF MODEL TABLE
       LXI	D,RSTBL
NXT:   LDAX	D
       MOV	M,A
       INX	H
       INX	D
       MVI	A,EOT
       CMP	L
       JNZ	NXT
       LXI	H,INIT
       SHLD START+1
       JMP	START
```

NINITはRSTBLからEOTまでに書かれている機械語サブルーチン群を0008H番地へコピーする。こうすることで、これらのサブルーチンはi8080のRST命令で呼び出すことができる。RST命令は1バイトで表現されるので、CALL命令(3バイト)で呼び出すよりもメモリの消費は少なくてすむ。

機械語サブルーチン群のコピーが終わったら、機械語プログラム領域の最初の命令のオペランドをNINITからINITに書き換えて、もう一度プログラムエリアの最初の命令(JMP INITになっている)を実行する。

```
TXTBGN DS   1         ;TEXT SAVE AREA BEGINS 
MSG1   DB   7FH,7FH,7FH,'SHERRY BROTHERS TINY BASIC VER. 3.1',0DH 
INIT   MVI  A,0FFH
       STA  OCSW      ;TURN ON OUTPUT SWITCH 
       MVI  A,0CH     ;GET FORM FEED 
       RST  2         ;SEND TO CRT 
PATLOP SUB  A         ;CLEAR ACCUMULATOR
       LXI  D,MSG1    ;GET INIT MESSAGE
       CALL PRTSTG    ;SEND IT
```

INITでは、まず起動メッセージ(SHERRY BROTHERS TINY BASIC...)を表示する。

```
LSTRAM LDA  7         ;GET FBASE FOR TOP
       STA  RSTART+2
       DCR  A         ;DECREMENT FOR OTHER POINTERS
       STA  SS1A+2    ;AND FIX THEM TOO
       STA  TV1A+2
       STA  ST3A+2
       STA  ST4A+2
       STA  IP3A+2
       STA  SIZEA+2
       STA  GETLN+3
       STA  PUSHA+2
```

次にRSTART(メイン)の最初の命令(LXI SP, STACK)のオペランドの上位バイトを書き換え、2000Hから0700Hへ変更する(LSTRAM)。

同様にSS1A/TV1A/ST3A/ST4A/IP3A/SIZEA/GETLN/PUSHAに含まれるVARBGN/TXTEND/BUFFER/STKLMTの上位バイトを06Hに書き換える。

整理すると、下記の表のようになる。BUFENDは0F87Hのままだが、下位バイトしか比較には用いないので問題ない。

|        | STACK | STKLMT | VARBGN | TXTEND | BUFFER | BUFEND |
|--------|-------|--------|--------|--------|--------|--------|
| 変更前 | 2000H | 0FAFH  | 0F00H  | 0F00H  | 0F37H  | 0F87H  |
| 変更後 | 0700H | 06AFH  | 0600H  | 0600H  | 0637H  | 0F87H  |

```
       LXI  H,ST1     ;GET NEW START JUMP
       SHLD START+1   ;AND FIX IT
       JMP  ST1
```

最後にNINITと同様に、機械語プログラム領域の最初の命令(JMP INIT)のオペランドを書き換え、JMP ST1とし、今度はST1へ直接ジャンプする。

ここまでで使用した、起動メッセージ文字列、INIT、NINIT、RST領域へコピーされた機械語プログラムは今後使用されることはないので、この領域は、BASICのプログラムテキスト領域や配列変数領域として最利用される。

### メインループ

RSTARTはTiny Basicインタープリタのメイン処理部である。エラーが発生した際もRSTARTへ戻ってくる。

```
RSTART LXI  SP,STACK  ;SET STACK POINTER
ST1    CALL CRLF      ;AND JUMP TO HERE
       LXI  D,OK      ;DE->STRING
       SUB  A         ;A=0 
       CALL PRTSTG    ;PRINT STRING UNTIL 0DH
       LXI  H,ST2+1   ;LITERAL 0 
       SHLD CURRNT    ;CURRNT->LINE # = 0
ST2    LXI  H,0 
       SHLD LOPVAR
       SHLD STKGOS
```

ST1からメインループが開始される。

ST1では画面に"OK"と印字し、CURRNTを0に設定する。CURRNTは現在行番号ポインタで行番号(0)そのものを代入するのではなく、0が入っているメモリ位置を代入する。

ST2ではFOR文で使用するループ変数(LOPVAR)、GOSUB文で使用するスタックポインタ保管変数(STKGOSを0に初期化する。

```
ST3    MVI  A,76Q     ;PROMPT '>' AND
       CALL GETLN     ;READ A LINE 
       PUSH D         ;DE->END OF LINE 
```

ST3ではプロンプト('>')を表示し、GETLNを呼び出してユーザーから入力を受け付ける。

GETLNから戻るとDEレジスタは入力の最後を指している。DEレジスタはコマンドの解析を行う際に使用するので、DEレジスタをスタックへ保存しておく。

```
ST3A   LXI  D,BUFFER  ;DE->BEGINNING OF LINE 
       CALL TSTNUM    ;TESt IFF IT IS A NUMBER
       RST  5 
       MOV  A,H       ;HL=VALUE OF THE # OR
       ORA  L         ;0 IFF NO # WAS FOUND 
       POP  B         ;BC->END OF LINE 
       JZ   DIRECT
```

DEレジスタを入力バッファの先頭に設定し、入力の解析を行う。

DEレジスタは、テキストポインタとして使われることが多く、ここでは入力バッファを指しているが、Tiny Basicプログラムを実行する際は、プログラムテキスト領域内の文を指している。

入力の先頭が数値でなければコマンドが入力されたと解釈し、ダイレクト実行を行う。ダイレクト実行が終わると、RSTARTへ戻る。

入力の先頭が数値であれば、行番号つきのステートメントであると解釈し、プログラムテキスト領域へコピーする。

GETLNを呼び出したあとにスタックに保存した入力の末尾位置をBCレジスタへ読み出しておく。この値は後で入力されたステートメントの長さを計算するために使用する。

```
       DCX  D         ;BACKUP DE AND SAVE
       MOV  A,H       ;VALUE OF LINE # THERE 
       STAX D 
       DCX  D 
       MOV  A,L 
       STAX D 
```

DEレジスタはステートメントの先頭を指しているので、その前の2バイトにHLレジスタに格納されている行番号を保管する。

DEレジスタが指すこの行番号の位置から、BCレジスタが指す入力の末尾までをプログラムテキスト領域へ挿入する。

```
       PUSH B         ;BC,DE->BEGIN, END 
       PUSH D 
       MOV  A,C 
       SUB  E 
       PUSH PSW       ;A=# OF BYTES IN LINE
```

挿入する領域の先頭と末尾をスタックに保存し、この領域の長さを計算する。入力バッファの長さは80バイトなので、下位バイトのみの演算(C-E)でよい。計算結果もスタックに保存しておく。

```
       CALL FNDLN     ;FIND THIS LINE IN SAVE
       PUSH D         ;AREA, DE->SAVE AREA 
       JNZ  ST4       ;NZ:NOT FOUND, INSERT
```

HLレジスタにはユーザが入力した行番号が入っている。FNDLNを呼び出して同じ行番号をもつステートメントをプログラムテキスト領域内で検索する。

FNDLN終了後のDEレジスタをスタックに保存する。行番号を持つステートメントが見つかっていれば、その行の先頭、見つかっていなければ、次の行の先頭を指している。

入力された文と同じ行番号を持つステートメントが存在していなければ、FNDLN終了後のDEレジスタの位置に入力された文を挿入する。
```
       PUSH D         ;Z:FOUND, DELETE IT
       CALL FNDNXT    ;FIND NEXT LINE
;*                                       DE->NEXT LINE 
       POP  B         ;BC->LINE TO BE DELETED
       LHLD TXTUNF    ;HL->UNFILLED SAVE AREA
       CALL MVUP      ;MOVE UP TO DELETE 
       MOV  H,B       ;TXTUNF->UNFILLED AREA 
       MOV  L,C 
       SHLD TXTUNF    ;UPDATE
```

同じ行番号を持つステートメントが見つかった場合、削除範囲を決定するために次の行を探す。

この段階でBCレジスタに削除する行の先頭、DEレジスタに削除する行の次の行の先頭位置が入っている。

次のMVUPはDE〜HLレジスタで示される範囲をBCレジスタから開始される領域に移動する。これでBC〜DEレジスタの範囲にある行が消去される。

BCレジスタの内容を(一旦HLに入れて)TXTUNFに代入することで、プログラムテキストの終了位置を更新する。
```
ST4    POP  B         ;GET READY TO INSERT 
       LHLD TXTUNF    ;BUT FIRT CHECK IF
       POP  PSW       ;THE LENGTH OF NEW LINE
       PUSH H         ;IS 3 (LINE # AND CR)
       CPI  3         ;THEN DO NOT INSERT
       JZ   RSTART    ;MUST CLEAR THE STACK
```

FNDLN呼び出し後にスタックに保存したテキストの挿入位置をBCレジスタに入れる。

挿入処理前にメモリの空き容量をチェックする。FNDLN呼び出し前にスタックに保存した行の長さをAレジスタに読み込む。

現在(挿入前)のプログラムテキストの終了位置をスタックに保存しておく。

行の長さが3(行番号2バイトと改行コード1バイト)であれば、空行であるとみなし、RSTARTへ戻り、スタックをクリアする。
```
       ADD  L         ;COMPUTE NEW TXTUNF
       MOV  L,A 
       MVI  A,0 
       ADC  H 
       MOV  H,A       ;HL->NEW UNFILLED AREA 
ST4A   LXI  D,TXTEND  ;CHECK TO SEE IF THERE 
       RST  4         ;IS ENOUGH SPACE 
       JNC  QSORRY    ;SORRY, NO ROOM FOR IT 
```

プログラムテキストの終了位置を計算する。その結果がプログラムテキスト領域の最大範囲を超えていたら、入力エラーとし、QSORRYへジャンプする。
```
       SHLD TXTUNF    ;OK, UPDATE TXTUNF 
       POP  D         ;DE->OLD UNFILLED AREA 
       CALL MVDOWN
       POP  D         ;DE->BEGIN, HL->END
       POP  H 
       CALL MVUP      ;MOVE NEW LINE TO SAVE 
       JMP  ST3       ;AREA
```

テキストの挿入が可能であることがわかったので、プログラムテキスト終了位置を更新する。

スタックから現在(挿入前)のプログラムテキスト終了位置を取り出す。

MVDOWNを呼び出し、文を書き込むスペースを確保する。

挿入する領域の範囲をスタックから取り出し、DEレジスタに開始位置、HLレジスタに終了位置を設定する。

MVUPを呼び出し、DE〜HLレジスタで示される範囲をBCレジスタで示される挿入位置へコピーする。

ST3へ戻り、新たな入力を待つ。

### コマンド実行

コマンド/文の実行は8つのコマンドテーブル(TAB1からTAB8)を用いて行われる。
```
TAB1   EQU  $         ;DIRECT COMMANDS 
       DB   'LIST'
       DB   LIST SHR 8 + 128,LIST AND 0FFH
```

コマンドテーブルの各項目はコマンド名を示す文字列とコマンドの処理内容が書かれているアドレスからなっている。

コマンドの処理内容が書かれているアドレスについては、アドレスそのものが書かれているわけではない。上位バイト、下位バイトの順序を逆に配置してあり、またアドレスの上位バイトの最上位ビットが1に設定してある。
  - ASCII文字の最上位ビットは0であるため、コマンドテーブル内で最上位ビットが1であるようなデータは、アドレスの一部である。また、Tiny Basicインタープリタは0600Hよりも下位のアドレスに配置してあるため、上位アドレスの最上位ビットが1になることはない。従って、テーブルを先頭から１バイトずつ見ていったときに、最上位ビットが1であるバイトが見つかったら、そのバイトと次のバイトはアドレスを構成することがわかる。
  - TAB1はダイレクト実行用のコマンドテーブルである。TAB1にはデフォルト動作が定義されていないため、TAB1内にコマンドが見つからなければ、そのままTAB2を検索する。
  - TAB2はステートメント用のコマンドテーブルである。このテーブルに含まれる一部のコマンドはダイレクト実行が可能である。
  - TAB4は関数用のコマンドテーブルである。
  - TAB5はFOR文用のコマンドテーブルで、'TO'が定義されている。
  - TAB6もFOR文用のコマンドテーブルで、'STEP'が定義されている。
  - TAB8は関係演算子が定義されている。

```
DIRECT LXI  H,TAB1-1  ;*** DIRECT ***
```

DIRECTは行番号が入力されていない場合に実行される。コマンドテーブルとしてTAB1を選択し、そのままEXECを実行する。

```
EXEC   EQU  $         ;*** EXEC ***
EX0    RST  5         ;IGNORE LEADING BLANKS 
       PUSH D         ;SAVE POINTER
```

最初に空白文字をスキップし、ステートメントの先頭位置をスタックに保存する。
```
EX1    LDAX D         ;IFF FOUND '.' IN STRING
       INX  D         ;BEFORE ANY MISMATCH 
       CPI  56Q       ;WE DECLARE A MATCH
       JZ   EX3 
       INX  H         ;HL->TABLE 
       CMP  M         ;IFF MATCH, TEST NEXT 
       JZ   EX1 
```

EX1は文字列比較ループの先頭になる。

ステートメント文字列に'.'が現れれば、コマンドテーブルの検索を中断し、比較中のコマンドテーブル項目を選択してループを抜ける(EX3へジャンプ)。

'.'が現れなければ、文字比較に失敗するまで文字列比較を続ける。
```
       MVI  A,177Q    ;ELSE, SEE IFF BIT 7
       DCX  D         ;OF TABLEIS SET, WHICH
       CMP  M         ;IS THE JUMP ADDR. (HI)
       JC   EX5       ;C:YES, MATCHED
```

文字比較に失敗したら、コマンドテーブル側の文字の最上位ビットが立っているかどうか調べる。最上位ビットが立っていれば、その文字はアドレスデータを表している。つまり、ステートメント文字列の検索に成功したとして、EX5へジャンプする。

```
EX2    INX  H         ;NC:NO, FIND JUMP ADDR.
       CMP  M 
       JNC  EX2 
       INX  H         ;BUMP TO NEXT TAB. ITEM
       POP  D         ;RESTORE STRING POINTER
       JMP  EX0       ;TEST AGAINST NEXT ITEM
```

ステートメント文字列比較に失敗した場合、アドレスデータが見つかるまで読み飛ばす(EX2のループ)。

アドレスデータが見つかったら、HLレジスタをインクリメントし、次のコマンドテーブル項目を指すようにする。

その後、スタックから入力されたステートメントの先頭位置を復元し、次のコマンドテーブル項目に対し、文字列比較を開始する。
```
EX3    MVI  A,177Q    ;PARTIAL MATCH, FIND 
EX4    INX  H         ;JUMP ADDR., WHICH IS
       CMP  M         ;FLAGGED BY BIT 7
       JNC  EX4
```

部分マッチでフープを抜けた場合、アドレスデータが見つかるまでコマンドテーブルを読み飛ばす(EX4のループ)。
```
EX5    MOV  A,M       ;LOAD HL WITH THE JUMP 
       INX  H         ;ADDRESS FROM THE TABLE
       MOV  L,M 
       ANI  177Q      ;MASK OFF BIT 7
       MOV  H,A 
       POP  PSW       ;CLEAN UP THE GABAGE 
       PCHL           ;AND WE GO DO IT 
```

アドレスデータをコマンドテーブルから読み出す。上位バイトにはアドレスデータであることを示すフラグが立ててあるので、このフラグを取り除いてアドレス情報を復元し、HLレジスタにセットする。

スタックにはまだ入力ステートメントの先頭アドレスが残っているので、これを読み捨ててから、PCHL命令でHLレジスタの内容をプログラムカウンタにセットすることで、ステートメントに対応するアドレスにジャンプする。
## ダイレクトコマンド

Basic処理系では、プログラム外から入力され、Tiny Basic処理系に命令を与えるものをコマンドと呼んでいる。一方でプログラムの中の文をステートメントと呼ぶ。
### RUN

RUNコマンドはプログラムテキスト領域に保管されているプログラムの実行を行う。

RUNコマンド処理部は、RUNNXL(次の行を実行)、RUNTSL(この行を実行)、RUNSML(同じ行を実行)の3部から構成される。
```
RUN    CALL ENDCHK    ;*** RUN(CR) *** 
       LXI  D,TXTBGN  ;FIRST SAVED LINE
```

RUNコマンドの処理が開始されると、最初にENDCHKで"RUN"文字列のあとにパラメータ文字列が存在しないことを確認する。ここまではDEレジスタは入力バッファを指している。

DEレジスタにプログラムテキスト領域の先頭アドレスをセットし、続けてRUNNXLを実行する。
```
RUNNXL LXI  H,0       ;*** RUNNXL ***
       CALL FNDLNP    ;FIND WHATEVER LINE #
       JC   RSTART    ;C:PASSED TXTUNF, QUIT 
```

RUNNXLでは、DEが指し示すプログラムバッファ領域を行番号0をFNDLNPで検索する。行番号が0なのでFNDLNは、次の行の行番号の位置をDEレジスタにセットして戻る。

もし、次の行がなければ、RSTARTへ戻る。
```
RUNTSL XCHG           ;*** RUNTSL ***
       SHLD CURRNT    ;SET 'CURRNT'->LINE #
       XCHG 
       INX  D         ;BUMP PASS LINE #
       INX  D 
```

RUNTSLはDEが指す行番号位置をCURRNTにセットする。

DEレジスタを2つ進め、行番号を読み飛ばす。
```
RUNSML CALL CHKIO     ;*** RUNSML ***
       LXI  H,TAB2-1  ;FIND COMMAND IN TAB2
       JMP  EXEC      ;AND EXECUTE IT
```

RUNSMLはCHKIOでキー入力を処理したあと、コマンドテーブルをTAB2にセットし、EXECへジャンプし、ステートメントを実行する。
### LIST

LISTコマンドは"LIST"または"LIST 行番号"の形で実行される。

単に"LIST"と入力された場合、プログラムテキスト領域に保管されたプログラムをすべて印字する。

"LIST 行番号"の場合は、行番号以降のプログラムを印字する。
```
LIST   CALL TSTNUM    ;TEST IFF THERE IS A #
       CALL ENDCHK    ;IFF NO # WE GET A 0
       CALL FNDLN     ;FIND THIS OR NEXT LINE
```

このルーチンが呼び出されたときは、DEレジスタは"LIST"のあとを指している。

最初にTSTNUMを呼び出し、DEレジスタが指している文字列が行番号かどうか確認する。TSTNUMはHLレジスタに行番号、Bに行番号の桁数を入れて戻る。

FNDLNを呼び出し、HLレジスタに入った行番号を持つステートメントを検索する。
```
LS1    JC   RSTART    ;C:PASSED TXTUNF 
       CALL PRTLN     ;PRINT THE LINE
       CALL CHKIO     ;STOP IFF HIT CONTROL-C 
       CALL FNDLNP    ;FIND NEXT LINE
       JMP  LS1       ;AND LOOP BACK 
```

もしステートメントが見つからなければ、RSTART(メインループ)へ戻る(LS1)。

ステートメントが見つかれば、行番号とステートメントを印字し、FNDLNPを呼び出して次の行を検索する。

LS1へ戻り、FNDLNPの呼び出しでステートメントが見つかっていれば、また行番号とステートメントを印字する。
### NEW

プログラムテキスト領域内に保管されたプログラムを消去する。
```
NEW    CALL ENDCHK    ;*** NEW(CR) *** 
       LXI  H,TXTBGN
       SHLD TXTUNF
```

実際にはTXTUNF(使用中のプログラムテキスト領域の末尾)にTXTBGN(プログラムテキスト領域の先頭)を代入するのみである。

続いてSTOPコマンドの処理を実行することでRSTARTへ戻る。
### BYE

BYEコマンドには機械語コードが存在しない。
```
       DB   'BYE',80H,0H   ;GO BACK TO CPM
```

コマンドテーブルの実行アドレス部には0番地が指定してある。つまりBYEコマンドは0番地へジャンプ、すなわち、CP/Mへ戻る(ウォームブート)。
### LOAD
```
DLOAD  RST  5         ;IGNORE BLANKS
       PUSH H         ;SAVE H
       CALL FCBSET    ;SET UP FILE CONTROL BLOCK
```

IGNBLK(RST 5)を呼び出して空白文字を取り除き、HLレジスタをスタックに退避したあと、ファイルの読み込みに必要なFCBブロックを作成する。
```
       PUSH D         ;SAVE THE REST
       PUSH B         
       LXI  D,FCB     ;GET FCB ADDRESS
       MVI  C,OPEN    ;PREPARE TO OPEN FILE
       CALL CPM       ;OPEN IT
       CPI  0FFH      ;IS IT THERE?
       JZ   QHOW      ;NO, SEND ERROR
```

テキストポインタ(DEレジスタ。LOAD文の末尾を挿している)とBCレジスタをスタックに退避する。DEレジスタにFCBのアドレスを設定し、CP/Mのシステムコールを呼び出してファイルを開く。

ファイルが開けなければ、エラー処理(QHOW)を行う。
```
       XRA  A         ;CLEAR A
       STA  FCB+32    ;START AT RECORD 0
```

FCBのCR(Current Record)フィールドに0を入れる。
```
       LXI  D,TXTUNF  ;GET BEGINNING
LOAD   PUSH D         ;SAVE DMA ADDRESS
       MVI  C,SETDMA  ;
       CALL CPM       ;SET DMA ADDRESS
```

テキストポインタ(DEレジスタ)に未使用領域の先頭アドレス(TXTUNF)を設定する。これが最初のDMAアドレスになる。

DMAアドレスをスタックに保存し、CP/Mのシステムコールを呼び出してDMAアドレスを設定する。
```
       MVI  C,READD   ;
       LXI  D,FCB
       CALL CPM       ;READ SECTOR
       CPI  1         ;DONE?
       JC   RDMORE    ;NO, READ MORE
       JNZ  QHOW      ;BAD READ
```

1ブロック読み込む。ファイルの読み込みが完了していなければ(戻り値が1)、RDMOREへジャンプする。

戻り値が負であれば、エラー処理(QHOW)を行う。
```
       MVI  C,CLOSE
       LXI  D,FCB 
       CALL CPM       ;CLOSE FILE
       POP  D         ;THROW AWAY DMA ADD.
       POP  B         ;GET OLD REGISTERS BACK
       POP  D
       POP  H
       RST  6         ;FINISH
```

ファイルの読み込みが正しく終わった場合、CP/Mのシステムコールを呼び出し、ファイルを閉じる。

スタックに積んであるDMAアドレスを廃棄し、BCレジスタ、DEレジスタ、HLレジスタを復元する。

FINISH(RST 6)を呼び出し、処理を完了する。
```
RDMORE POP  D         ;GET DMA ADDRESS
       LXI  H,80H     ;GET 128
       DAD  D         ;ADD 128 TO DMA ADD.
       XCHG           ;PUT IT BACK IN D
       JMP  LOAD      ;AND READ SOME MORE
```

まだ読み込むデータがある場合、DMAアドレスに80Hを足してLOADに戻る。
### SAVE
```
DSAVE  RST  5         ;IGNORE BLANKS
       PUSH H         ;SAVE H
       CALL FCBSET    ;SETUP FCB
       PUSH D
       PUSH B         ;SAVE OTHERS
```

FCBブロックを作成するところまではDLOADと同じである。
```
       LXI  D,FCB
       MVI  C,DELETE
       CALL CPM       ;ERASE FILE IF IT EXISTS
       LXI  D,FCB  
       MVI  C,MAKE
       CALL CPM       ;MAKE A NEW ONE
       CPI  0FFH      ;IS THERE SPACE?
       JZ   QHOW      ;NO, ERROR
```

"DLOAD"ではファイルをオープンしたが、最初に"DSAVE"ではファイルを削除し、同じ名前でファイルを作成する。ファイル作成に失敗した場合、エラー処理(QHOW)を行う。
```
       XRA  A         ;CLEAR A
       STA  FCB+32    ;START AT RECORD 0
       LXI  D,TXTUNF  ;GET BEGINNING
SAVE   PUSH D         ;SAVE DMA ADDRESS
       MVI  C,SETDMA  ;
       CALL CPM       ;SET DMA ADDRESS
```

"LOAD"と同様にDMAアドレスを設定する。
```
       MVI  C,WRITED
       LXI  D,FCB 
       CALL CPM       ;WRITE SECTOR
       ORA  A         ;SET FLAGS
       JNZ  QHOW      ;IF NOT ZERO, ERROR
```

CP/Mのシステムコールでファイルを書き込む。書込みに失敗したら、エラー処理(QHOW)を行う。
```
       POP  D         ;GET DMA ADD. BACK
       LDA  TXTUNF+1  ;AND MSB OF LAST ADD.
       CMP  D         ;IS D SMALLER?
       JC   SAVDON    ;YES, DONE
       JNZ  WRITMOR   ;DONT TEST E IF NOT EQUAL
```

DMAアドレスの上位バイトと、未使用テキスト領域の上位バイトを比較する。DMAアドレスのほうが大きければ、保存は終了していると判断し、SAVDONへジャンプする。

一致していなければ(DMAアドレスのほうが小さければ)、まだ書き込むデータがあると判断し、WRITMORへジャンプする。
```
       LDA  TXTUNF    ;IS E SMALLER?
       CMP  E
       JC   SAVDON    ;YES, DONE
```

下位バイトを比較する。未使用テキスト領域のアドレスのほうが小さければ、保存は終了したと判断し、SAVDONへジャンプする。
```
WRITMOR LXI  H,80H 
        DAD  D         ;ADD 128 TO DMA ADD.
        XCHG           ;GET IT BACK IN D
        JMP  SAVE      ;WRITE SOME MORE
```

"RDMORE"と同様にDMAアドレスに80Hを加算し、SAVEへジャンプする。
```
SAVDON MVI  C,CLOSE
       LXI  D,FCB 
       CALL CPM       ;CLOSE FILE
       POP  B         ;GET REGISTERS BACK
       POP  D
       POP  H
       RST  6         ;FINISH
```

保存が終わったら、CP/Mのシステムコールを呼び出し、ファイルを閉じる。

BCレジスタ、DEレジスタ、HLレジスタを復元する。LOADの時とは異なり、終了判定の際にFCBをスタックから取り出しているので、FCBの廃棄を行う必要はない。

FINISH(RST 6)を呼び出して終了処理を行う。
### FCBSET

FCB(File Control Block)は、CP/Mでファイルを操作するときに用いられるデータ構造であり、33〜36バイトからなっている。規定のFCB領域は005CHとなっている。
```
FCBSET LXI  H,FCB     ;GET FILE CONTROL BLOCK ADDRESS
       MVI  M,0       ;CLEAR ENTRY TYPE
```

まずFCBの最初のバイト(ドライブレコード)をクリアする。
```
FNCLR  INX  H         ;NEXT LOCATION
       MVI  M,' '     ;CLEAR TO SPACE
       MVI  A,FCB+8 AND 255
       CMP  L         ;DONE?
       JNZ  FNCLR     ;NO, DO IT AGAIN
```

次に2バイトから9バイト目(ファイル名)をスペースで消去する。
```
       INX  H         ;NEXT
       MVI  M,'T'     ;SET FILE TYPE TO 'TBI'
       INX  H
       MVI  M,'B'
       INX  H
       MVI  M,'I'
```

10バイト目から12バイト目までにファイルの種類(拡張子)を記入する。Tiny Basicプログラムの拡張子は"TBI"である。
```
EXRC   INX  H         ;CLEAR REST OF FCB
       MVI  M,0
       MVI  A,FCB+15 AND 255
       CMP  L         ;DONE?
       JNZ  EXRC      ;NO, CONTINUE
```

13〜15バイト目を0クリアする。
```
       LXI  H,FCB+1   ;GET FILENAME START
FN     LDAX D         ;GET CHARACTER
       CPI  0DH       ;IS IT A 'CR'
       RZ             ;YES, DONE
```

FNではFCBの2バイト目にテキストポインタ(DEレジスタ)が示す文字列(ファイル名)をコピーする。

テキストポインタが指す文字をAレジスタに読み込む。Aレジスタが復帰コード(0DH)であれば、呼び出し元に戻る。
```
       CPI  '!'       ;LEGAL CHARACTER?
       JC   QWHAT     ;NO, SEND ERROR
       CPI  '['       ;AGAIN
       JNC  QWHAT     ;DITTO
       MOV  M,A        ;SAVE IT IN FCB
       INX  H         ;NEXT
       INX  D
       MVI  A,FCB+9 AND 255
       CMP  L         ;LAST?
       JNZ  FN        ;NO, CONTINUE
       RET            ;TRUNCATE AT 8 CHARACTERS
```

文字(Aレジスタ)が'!'以上、'['より小さいことを確認する。ファイル名に使用する文字はこの範囲でなければならない。この範囲に入っていなければ、エラー処理(QWHAT)を行う。範囲内であれば、FCBにコピーする。

FCBのファイル名領域がいっぱいになったら、呼び出し元に戻る。この時ファイル名は8文字で切り捨てられる。
## ステートメント
### LET

"LET"コマンドは変数に値を代入する。

代入文は','で区切って複数記述することができる。

また、"LET"は省略することが可能である。コマンドテーブルのDEFLTはLETコマンドを実行する。
```
LET    CALL SETVAL    ;*** LET *** 
       RST  1         ;SET VALUE TO VAR. 
       DB   ',' 
       DB   3Q
       JMP  LET       ;ITEM BY ITEM
LT1    RST  6         ;UNTIL FINISH
```

最初にSETVALを呼び出して代入文を解析し、変数に値を設定する。

TSTC(RST 1)を呼び出して、次の文字が','であるかどうかを確認し、','であれば次の代入文を解析する。','でなければ、FINISH(RST 6)を呼び出し、次のステートメントの解析に移る。
### IF

'IF'コマンドのあとには、条件式が続く。

条件式が真であれば、その後ろに続く複数のコマンドを実行する。
```
IFF    RST  3         ;*** IFF ***
       MOV  A,H       ;IS THE EXPR.=0? 
       ORA  L 
       JNZ  RUNSML    ;NO, CONTINUE
       CALL FNDSKP    ;YES, SKIP REST OF LINE
       JNC  RUNTSL
       JMP  RSTART
```

最初にEXPR(RST 3)を呼び出し、条件式を解析する。EXPRはHLレジスタに結果を返す。

HLが0かどうかを確認し、0でなければ、後続のコマンドを実行する(RUNSML)。

HLが0であれば、行の残りを読み飛ばす。

読み飛ばしたあと、まだ次の行があれば、次の行を実行する(RUNTSL)。次の行がなければメインループに戻る。
### GOTO

'GOTO'コマンドは'GOTO 式'の形を取り、式の値を持つ行番号へジャンプする。
```
GOTO   RST  3         ;*** GOTO EXPR *** 
       PUSH D         ;SAVE FOR ERROR ROUTINE
       CALL ENDCHK    ;MUST FIND A 0DH
       CALL FNDLN     ;FIND THE TARGET LINE
       JNZ  AHOW      ;NO SUCH LINE #
       POP  PSW       ;CLEAR THE "PUSH DE" 
       JMP  RUNTSL    ;GO DO IT
```

最初にEXPR(RST 3)で式を評価する。結果はHLレジスタに入っている。

エラー処理時にエラー発生箇所を印字するため、DEレジスタをスタックに保存しておく。

行の終わりかどうかを確認する(ENDCHK)。行の終わりでなければ、エラーになる。

FNDLNでHLレジスタに入っている行番号をもつ文を探す。文が見つからなければエラーになる(AHOW)。行番号が見つかれば、DEレジスタが見つけた行を示している。

エラーが発生しなかった場合、スタックに保存した古いDEレジスタの内容を破棄し(POP PSW)、DEレジスタが示す行を実行する(RUNTSL)。
### GOSUB

'GOSUB'コマンドは'GOSUB 式'の形を取り、式の値を持つ行番号へジャンプする。

ただし、'GOTO'コマンドと違って、'RETURN'文に遭遇すると、GOSUB文の次の行の実行を始める。
```
GOSUB  CALL PUSHA     ;SAVE THE CURRENT "FOR"
       RST  3         ;PARAMETERS
       PUSH D         ;AND TEXT POINTER
       CALL FNDLN     ;FIND THE TARGET LINE
       JNZ  AHOW      ;NOT THERE. SAY "HOW?" 
       LHLD CURRNT    ;FOUND IT, SAVE OLD
       PUSH H         ;'CURRNT' OLD 'STKGOS' 
       LHLD STKGOS
       PUSH H 
       LXI  H,0       ;AND LOAD NEW ONES 
       SHLD LOPVAR
       DAD  SP
       SHLD STKGOS
       JMP  RUNTSL    ;THEN RUN THAT LINE
```

まず、FOR文の変数をスタックに退避する。

次にEXPR(RST 3)でジャンプ先の行番号を計算するところから、行の検索、エラー時にAHOWへ飛ぶところまではGOTOステートメントと同じ。

'RETURN'で戻ってきたときに備えて現在の行番号をスタックに保存する。

また、GOSUB文が入れ子になっている場合、STKGOSは使用中なので、現在のSTKGOSをスタックに退避する。

ループ変数(LOPVAR)に0(NULL)を設定し、STKGOSに現在のスタックポインタを入れる。

RUNTSLでDEレジスタが示す行を実行する。
### RETURN

"RETURN"ステートメントは、直前に実行されたGOSUBステートメントの直後の行へジャンプする。
```
RETURN CALL ENDCHK    ;THERE MUST BE A 0DH
       LHLD STKGOS    ;OLD STACK POINTER 
       MOV  A,H       ;0 MEANS NOT EXIST 
       ORA  L 
       JZ   QWHAT     ;SO, WE SAY: "WHAT?" 
       SPHL           ;ELSE, RESTORE IT
       POP  H 
       SHLD STKGOS    ;AND THE OLD 'STKGOS'
       POP  H 
       SHLD CURRNT    ;AND THE OLD 'CURRNT'
       POP  D         ;OLD TEXT POINTER
       CALL POPA      ;OLD "FOR" PARAMETERS
       RST  6         ;AND WE ARE BACK HOME
```

まず、"RETURN"のあとに何もないことを確認する。

STKGOSを確認する。GOSUBステートメントはSTKGOSにスタックポインタの値を保存するので、値が0であれば、エラーである。

値が0でなければ、スタックポインタを復元する。

スタックから一つ前のSTKGOS、CURRNTを復元する。

テキストポインタをスタックから復元し、DEレジスタに設定する。

FOR文の変数をスタックから取り出す。

FINISH(RST 6)を呼び出し、次のステートメントの解析に移る。
### REM

"REM"のあとに書かれた文は無視される。
```
REM    LXI  H,0Q      ;*** REM *** 
       DB   76Q 
;* 
IFF    RST  3         ;*** IFF ***
       MOV  A,H       ;IS THE EXPR.=0? 
       ORA  L 
       JNZ  RUNSML    ;NO, CONTINUE
       CALL FNDSKP    ;YES, SKIP REST OF LINE
```

まずHLレジスタに0を入れる。

次に"DB 76Q"が来るが、ここは少々わかりにくい。

76Q=3EHは"MVI A, byte"を意味している。"RST 3"は16進数でDFHとなるので、"MVI A, DFH"を実行している。ところがすぐに"MOV A, H"でAに0を代入するので"DB 76Q; RST 3"の２行は意味がない。

つまりREM文は条件式が0のIFステートメントに等しい。条件式が0なので、行末までスキップし、次の行があれば次の行を実行し、なければメインループに戻る。
### FOR〜NEXT

"FOR"ステートメントは"FOR 変数=式1 TO 式2 STEP 式3"の形式を取るが、"STEP"以降は省略可能である。

"NEXT"ステートメントは"NEXT 変数"の形を取る。
```
FOR    CALL PUSHA     ;SAVE THE OLD SAVE AREA
       CALL SETVAL    ;SET THE CONTROL VAR.
       DCX  H         ;HL IS ITS ADDRESS 
       SHLD LOPVAR    ;SAVE THAT 
       LXI  H,TAB5-1  ;USE 'EXEC' TO LOOK
       JMP  EXEC      ;FOR THE WORD 'TO' 
```

古いFOR文変数をスタックに保管する。

"FOR"キーワードのあとに続く"変数=式1"を解析し、変数に初期値(式1)を設定する。

HLレジスタは変数の上位バイトを指しているので、下位バイトを指すようにデクリメントする。

LOPVARに変数のアドレス(HLレジスタの内容)を保管する。

"TO"キーワードを検索するため、TAB5をコマンドテーブルとして選択し、EXECへジャンプする。
```
FR1    RST  3         ;EVALUATE THE LIMIT
       SHLD LOPLMT    ;SAVE THAT 
       LXI  H,TAB6-1  ;USE 'EXEC' TO LOOK
       JMP  EXEC      ;FOR THE WORD 'STEP'
```

"TO"キーワードが見つかったら、FR1へ制御が移る。

EXPR(RST 3)を実行し、式2を評価する。

式2をLOPLMTに保存する。

"STEP"キーワードを検索するためにTAB6をコマンドテーブルとして選択し、EXECへジャンプする。
```
FR2    RST  3         ;FOUND IT, GET STEP
       JMP  FR4 
FR3    LXI  H,1Q      ;NOT FOUND, SET TO 1 
FR4    SHLD LOPINC    ;SAVE THAT TOO 
```

"STEP"キーワードが見つかるとFR2へ制御が移り、式2を評価する。結果はHLレジスタに入る。

"STEP"キーワードが見つからなければFR3へ制御が移り、HLレジスタに1を設定する。

HLレジスタの内容をLOPINCへ保存する。
```
FR5    LHLD CURRNT    ;SAVE CURRENT LINE # 
       SHLD LOPLN 
       XCHG           ;AND TEXT POINTER
       SHLD LOPPT 
       LXI  B,12Q     ;DIG INTO STACK TO 
       LHLD LOPVAR    ;FIND 'LOPVAR' 
       XCHG 
       MOV  H,B 
       MOV  L,B       ;HL=0 NOW
       DAD  SP        ;HERE IS THE STACK 
```

LOPLN = CURRNT(現在の行番号)、LOPPT = テキストポインタとする。

BCレジスタ=000AHとする。Bレジスタは00Hであるので、"MOV H, B; MOV L, B"はHLレジスタ=0と同義である。

DEレジスタ=LOPVARとする。

"DAD SP"でHL=HL+SP=SPとなる。
```
       DB   76Q 
FR7    DAD  B         ;EACH LEVEL IS 10 DEEP 
       MOV  A,M       ;GET THAT OLD 'LOPVAR' 
       INX  H 
       ORA  M 
       JZ   FR8       ;0 SAYS NO MORE IN IT
       MOV  A,M 
       DCX  H 
       CMP  D         ;SAME AS THIS ONE? 
       JNZ  FR7 
       MOV  A,M       ;THE OTHER HALF? 
       CMP  E 
       JNZ  FR7 
       XCHG           ;YES, FOUND ONE
       LXI  H,0Q
       DAD  SP        ;TRY TO MOVE SP
       MOV  B,H 
       MOV  C,L 
       LXI  H,12Q 
       DAD  D 
       CALL MVDOWN    ;AND PURGE 10 WORDS
       SPHL           ;IN THE STACK
FR8    LHLD LOPPT     ;JOB DONE, RESTORE DE
       XCHG 
       RST  6         ;AND CONTINUE
```

"DB 76Q"は"REM"ステートメントのときと同じである。FR5からFR7へ制御が移るとき、FR7の最初の命令("DAD B")を無視する。ジャンプ命令でFR7へ制御が写った場合は、HL=HL+000AH=HL+10とする。

HLレジスタが指す位置(SP+10*n)の内容(古いLOPVAR)のチェックを行う。

古いLOPVARが0であれば、FR8へジャンプする。FR8ではLOPPT(テキストポインタ)をDEレジスタに復元し、FINISH(RST 6)を呼び出す。

古いLOPVARがDEレジスタ(現在のLOPVAR)と異なれば、FR7へ戻ってHLレジスタに10を足し、もう一つ古いLOPVARをチェックする。

古いLOPVARと現在のLOPVARが同じであれば、DE=HL、BC=SP、HL=HL+10とし、MVDOWNを呼び出して、古いFOR変数を消去する。
```
NEXT   RST  7         ;GET ADDRESS OF VAR. 
       JC   QWHAT     ;NO VARIABLE, "WHAT?"
       SHLD VARNXT    ;YES, SAVE IT
```

TSTV(RST 7)を呼び出し、キーワード"NEXT"の次の変数を確認してVARNXTに入れる。
```
NX0    PUSH D         ;SAVE TEXT POINTER 
       XCHG 
       LHLD LOPVAR    ;GET VAR. IN 'FOR' 
       MOV  A,H 
       ORA  L         ;0 SAYS NEVER HAD ONE
       JZ   AWHAT     ;SO WE ASK: "WHAT?"
       RST  4         ;ELSE WE CHECK THEM
       JZ   NX3       ;OK, THEY AGREE
       POP  D         ;NO, LET'S SEE 
       CALL POPA      ;PURGE CURRENT LOOP
       LHLD VARNXT    ;AND POP ONE LEVEL 
       JMP  NX0       ;GO CHECK AGAIN
```

LOPVAR=0であれば、FOR文無しでNEXTが呼ばれたと判断し、エラーとする。

VARNXTとLOPVARを比較し(COMP/RST 4)、一致しなければスタックからLOPVARを含むFOR変数一式を削除する。NX0へ戻り、同じことを繰り返す。
```
NX3    MOV  E,M       ;COME HERE WHEN AGREED 
       INX  H 
       MOV  D,M       ;DE=VALUE OF VAR.
       LHLD LOPINC
       PUSH H 
       DAD  D         ;ADD ONE STEP
       XCHG 
       LHLD LOPVAR    ;PUT IT BACK 
       MOV  M,E 
       INX  H 
       MOV  M,D 
       LHLD LOPLMT    ;HL->LIMIT 
       POP  PSW       ;OLD HL
       ORA  A 
       JP   NX1       ;STEP > 0
       XCHG 
```

VARNXTとLOPVARが一致した場合、NX3へ制御が移る。

VARNXT(=LOPVAR)が指す変数の内容をDEレジスタへ読み込む。

HLレジスタへLOPINCの値を読み込む。

HL=HL+DEとしてLOPVARの値を更新する。

HLレジスタへLOPLMTの値を読み込む。

スタックからLOPINCの値を読み出し、LOPINC>0ならNX1へジャンプする。そうでなければ、DE(LOPVARの内容)とHL(LOPLMT)を入れ替える。
```
NX1    CALL CKHLDE    ;COMPARE WITH LIMIT
       POP  D         ;RESTORE TEXT POINTER
       JC   NX2       ;OUTSIDE LIMIT 
       LHLD LOPLN     ;WITHIN LIMIT, GO
       SHLD CURRNT    ;BACK TO THE SAVED 
       LHLD LOPPT     ;'CURRNT' AND TEXT 
       XCHG           ;POINTER 
       RST  6 
NX2    CALL POPA      ;PURGE THIS LOOP 
       RST  6 
```

HLとDEを比較する。"NEXT"の最初でスタックに保存したテキストポインタを復元しておき、HL<DEならNX2へジャンプし、FOR文を終了する。

DE<HLならLOPLN(FORループ内の先頭の行番号)をCURRNTに設定し、DEレジスタにLOPPT(FORループ内の最初の文)を設定する。FINISH(RST 6)を呼び出してループ内のステートメントを実行する。
### STOP
"STOP"ステートメントは、プログラムの実行を中断し、メインループに戻る。
```
STOP   CALL ENDCHK    ;*** STOP(CR) ***
       JMP RSTART
```

"STOP"のあとに何も書かれていないことを確認する。書かれていればエラーになる。
### INPUT

"INPUT"ステートメントは、ユーザーに変数へ値を入力させる。
```
INPUT  EQU  $         ;*** INPUT *** 
IP1    PUSH D         ;SAVE IN CASE OF ERROR 
       CALL QTSTG     ;IS NEXT ITEM A STRING?
       JMP  IP2       ;NO
       RST  7         ;YES. BUT FOLLOWED BY A
       JC   IP4       ;VARIABLE?   NO. 
       JMP  IP3       ;YES.  INPUT VARIABLE
```

テキストポインタ(DEレジスタ)をスタックに退避する。

QTSTGを呼び出し、次のテキストが文字列であるかどうか確認する。QTSTGは次のテキストが文字列であった場合、"JMP IP2(3バイト)"をスキップした箇所に戻ってくる。文字列でなかった場合、"JMP IP2"を実行しIP2へジャンプする。

次のテキストが文字列である場合、TSTV(RST 7)を呼び出して、文字列のあとに変数が続くか確認する。変数でなければIP4へジャンプする。変数であればIP3へジャンプする。
```
IP2    PUSH D         ;SAVE FOR 'PRTSTG' 
       RST  7         ;MUST BE VARIABLE NOW
       JC   QWHAT     ;"WHAT?" IT IS NOT?
       LDAX D         ;GET READY FOR 'RTSTG'
       MOV  C,A 
       SUB  A 
       STAX D 
       POP  D 
       CALL PRTSTG    ;PRINT STRING AS PROMPT
       MOV  A,C       ;RESTORE TEXT
       DCX  D 
       STAX D 
```

"INPUT"ステートメントの引数が文字列ではなかった場合、PRTSTGでINPUTキーワードの次にくる文字を印字するが、その前にTSTV(RST 7)で次の文字が変数を表しているか確認する必要がある。TSTVはテキストポインタを更新するので、PRTSTGでの印字に備えて、テキストポインタ(DEレジスタ)をスタックに退避しておく。

TSTV(RST 7)で次の文字が変数を表しているか確認する。変数でなければエラー(QWHAT)とする。

変数であれば、HLレジスタには変数のアドレスが入る。変数名の次の文字をCレジスタに退避しておき、変数名の次の文字を0で上書きする。

テキストポインタを復元し(TSTVを呼び出す前なので、変数名を指している)、PRTSTGを呼び出して変数名を印字する。

0で上書きした箇所をもとの値に復元する。
```
IP3    PUSH D         ;SAVE IN CASE OF ERROR 
       XCHG 
       LHLD CURRNT    ;ALSO SAVE 'CURRNT'
       PUSH H 
       LXI  H,IP1     ;A NEGATIVE NUMBER 
       SHLD CURRNT    ;AS A FLAG 
       LXI  H,0Q      ;SAVE SP TOO 
       DAD  SP
       SHLD STKINP
       PUSH D         ;OLD HL
       MVI  A,72Q     ;PRINT THIS TOO
       CALL GETLN     ;AND GET A LINE
```

テキストポインタ(DEレジスタ)と行番号(CURRNT)をスタックに退避する。

IP1をエラー処理のためにCURRNTに保存する。IP1には'PUSH D(D5H)'が書かれているので、負の行番号を設定している。

STKINPにSPの値を保存する。

HLレジスタ(変数のアドレス)をスタックに保管する。

Aレジスタに':'を入れてGETLNを呼び出す。
```
IP3A   LXI  D,BUFFER  ;POINTS TO BUFFER
       RST  3         ;EVALUATE INPUT
       NOP            ;CAN BE 'CALL ENDCHK'
       NOP
       NOP
       POP  D         ;OK, GET OLD HL
       XCHG 
       MOV  M,E       ;SAVE VALUE IN VAR.
       INX  H 
       MOV  M,D 
       POP  H         ;GET OLD 'CURRNT'
       SHLD CURRNT
       POP  D         ;AND OLD TEXT POINTER
```

テキストポインタをGETLNの入力バッファに設定する。

EXPR(RST 3)を呼び出し、ユーザの入力を評価する。

EXPR(RST 3)のあとにNOPを3回呼び出しているが、代わりに'CALL ENDCHK(3バイト)'を入れてもよい。

変数のアドレスをDEレジスタに復元したあと、DEとHLを入れ替える。この時点でHLレジスタには変数のアドレス、DEレジスタにはユーザ入力の評価結果が入っている。

変数にDEレジスタの値を保管する。

行番号(CURRNT)を復元する。

テキストポインタをDEレジスタに復元する。
```
IP4    POP  PSW       ;PURGE JUNK IN STACK 
       RST  1         ;IS NEXT CH. ','?
       DB   ',' 
       DB   3Q
       JMP  IP1       ;YES, MORE ITEMS.
IP5    RST  6 
```

入力エラーに備えて、IP1でテキストポインタをスタックに保存した。この値がまだスタックに残っているので、AFレジスタへ取り出すことでこの値を廃棄する。

TSTC(RST 1)を呼び出し、次の文字が','であるかを確認する。','でなければ、IP5へジャンプし、FINISH(RST 6)を実行する。','であれば、IP1へジャンプし、新しい入力を取得する。
### PRINT

"PRINT"ステートメントは引数を印字する。
```
PRINT  MVI  C,6       ;C = # OF SPACES 
       RST  1         ;IFF NULL LIST & ";"
       DB   73Q 
       DB   6Q 
       CALL CRLF      ;GIVE CR-LF AND
       JMP  RUNSML    ;CONTINUE SAME LINE
```

Cレジスタに6を入れる。

TSTC(RST 1)を呼び出し、テキストポインタが';'を指しているかどうか確認する。';'であれば、CRLFを印字し、同じ行にあるPRINTステートメントの次の文を実行する。';'でなければPR2へジャンプする。
```
PR2    RST  1         ;IFF NULL LIST (CR) 
       DB   0DH
       DB   6Q
       CALL CRLF      ;ALSO GIVE CR-LF AND 
       JMP  RUNNXL    ;GO TO NEXT LINE 
```

TSTC(RST 1)を呼び出し、テキストポインタが復帰コード(0DH)を指しているかどうか確認する。復帰コードであれば、CRLFを印字し、次の行を実行する。
```
PR0    RST  1         ;ELSE IS IT FORMAT?
       DB   '#' 
       DB   5Q
       RST  3         ;YES, EVALUATE EXPR. 
       MOV  C,L       ;AND SAVE IT IN C
       JMP  PR3       ;LOOK FOR MORE TO PRINT
```

TSTC(RST 1)を呼び出し、テキストポインタが'#'を指しているかどうか確認する。'#'であれば、EXPR(RST 3)を呼び出し、'#'に続く式を評価する。結果をCレジスタに入れて、PR3にジャンプする。'#'でなければ、PR1へジャンプする。
```
PR1    CALL QTSTG     ;OR IS IT A STRING?
       JMP  PR8       ;IFF NOT, MUST BE EXPR. 
```

PR0で確認した文字が'#'でなかった場合、QTSTGを呼び出して文字列かどうか確認する。文字列であればQTSTGはPR3へ戻る。文字列でなければ、PR8を実行する。
```
PR3    RST  1         ;IFF ",", GO FIND NEXT
       DB   ',' 
       DB   6Q
       CALL FIN       ;IN THE LIST.
       JMP  PR0       ;LIST CONTINUES
PR6    CALL CRLF      ;LIST ENDS 
       RST  6 
```

PR0で確認した文字が'#'と式であった場合、次に','が続くか確認する。','であれば、FINを呼び出して次の文字が';'や復帰コードでないことを確認し、PR0へジャンプして次の要素を確認する。','でなければ、PR6へジャンプし、CRLFの印字とFINISH(RST 6)の呼び出しを行う。
```
PR8    RST  3         ;EVALUATE THE EXPR 
       PUSH B 
       CALL PRTNUM    ;PRINT THE VALUE 
       POP  B 
       JMP  PR3       ;MORE TO PRINT?
```

PR1での確認結果が文字列ではなかった場合、テキストポインタは式を指しているはずである。EXPR(RST 3)を呼び出して式を評価する。

PRTNUMを呼び出して式の評価結果を印字する。PRTNUMを呼び出すとBレジスタの内容が変更されるので、呼び出す前にスタックに退避しておく。

PR3へジャンプし、印字を継続する必要があるかを確認する。
## 式
### EXPR(RST 3)

EXPRは算術式、または論理式を評価する。
  - <EXPR>::=<EXPR2> OR <EXPR2><REL.OP.><EXPR2>
    - ここで<REL.OP.>はTAB8に含まれる関係演算子の一つであり、結果が真であれば1、偽であれば0になる。
  - <EXPR2>::=(+ OR -)<EXPR3>(+ OR -<EXPR3>)(....)
    - ここで()で囲まれる部分は省略可能であり、(....)は省略可能な繰り返しである。
  - <EXPR3>::=<EXPR4>(<* OR /><EXPR4>)(....)
  - <EXPR4>::=<VARIABLE> OR <FUNCTION> OR (<EXPR>)
    - <EXPR>は再帰的に定義されている。
    - 配列変数('@'で始まる)もインデックスとして<EXPR>を取ることができる。
    - 関数は<EXPR>を引数に取ることができる。
    - <EXPR4>は()内に<EXPR>を含んだ形にもできる。
```
       CALL EXPR2     ;*** EXPR OR RST 3 *** 
       PUSH H         ;EVALUATE AN EXPRESION 
       JMP  EXPR1     ;REST OF IT IS AT EXPR1
```

EXPR(RST 3)を呼び出すと、まずEXPR2を呼び出し、その結果をスタックへ保存する。その後、EXPR1へジャンプし、処理を継続する。
```
EXPR1  LXI  H,TAB8-1  ;LOOKUP REL.OP.
       JMP  EXEC      ;GO DO IT
```

EXPR1ではコマンドテーブルをTAB8にセットし、<EXPR2>のあとに関係演算子(<REL.OP.>)が続くかどうかを確認する。

関係演算子が続けば、対応する処理(XP11〜XP16)を実行し、関係演算子が続かなければ、XP17を実行する。
```
XP11   CALL XP18      ;REL.OP.">=" 
       RC             ;NO, RETURN HL=0 
       MOV  L,A       ;YES, RETURN HL=1
       RET
```

XP11〜XP16はほとんど同じなので、XP11(">=")のみを見ていく。

XP18を呼び出し、その結果に応じてHLレジスタに0、または1を設定する。

XP11はEXPR(RST 3)からJMP命令でEXPR1、EXPR1からJMP命令でEXEC、EXECからPCHL命令でXP11へと制御が移ってくるので、最後のRET命令で、EXPR(RST 3)の呼び出し元に戻る。
```
XP18   MOV  A,C       ;SUBROUTINE FOR ALL
       POP  H         ;REL.OP.'S 
       POP  B 
       PUSH H         ;REVERSE TOP OF STACK
       PUSH B 
       MOV  C,A 
```

XP18へはCALL命令で制御が移るため、スタックの最上位にはRETURN先のアドレスが入っている。スタックの上位２項目を入れ替え、関係演算子の左の項をスタックの最上位に移動させる。
```
       CALL EXPR2     ;GET 2ND <EXPR2> 
       XCHG           ;VALUE IN DE NOW 
       XTHL           ;1ST <EXPR2> IN HL 
       CALL CKHLDE    ;COMPARE 1ST WITH 2ND
       POP  D         ;RESTORE TEXT POINTER
       LXI  H,0Q      ;SET HL=0, A=1 
       MVI  A,1 
       RET
```

関係演算子の右の項<EXPR2>を評価し、XCHG命令でDEレジスタに入れる。このときDEレジスタの内容(テキストポインタ)はHLレジスタに入る。続いてXTHL命令でスタックの最上位のデータ、つまり関係演算子の左の項をHLレジスタへ入れ、代わりにHLレジスタに入っていたテキストポインタをスタックへ退避する。

HLレジスタとDEレジスタの内容を比較し、フラグレジスタを変化させる。

スタックからテキストポインタをDEレジスタに復元し、HLレジスタに0、Aレジスタに1を設定する。

XP18の呼び出し元は、フラグレジスタの値に応じて、HLレジスタに適切な値を設定する。
```
XP17   POP  H         ;NOT REL.OP. 
       RET            ;RETURN HL=<EXPR2> 
```

関係演算子ではなかった場合、XP17が呼び出される。XP17は先にスタックに積んだ演算結果(<EXPR2>)を取り出し、HLレジスタに設定してEXPR(RST 3)の呼び出し元に戻る。
```
EXPR2  RST  1         ;NEGATIVE SIGN?
       DB   '-' 
       DB   6Q
       LXI  H,0Q      ;YES, FAKE '0-'
       JMP  XP26      ;TREAT LIKE SUBTRACT 
```

負号で始まれば、HL=0としてXP26へジャンプする。
```
XP26   PUSH H         ;YES, SAVE 1ST <EXPR3> 
       CALL EXPR3     ;GET 2ND <EXPR3> 
       CALL CHGSGN    ;NEGATE
       JMP  XP24      ;AND ADD THEM
```

XP26ではHLレジスタの内容(0)をスタックに保存し、EXPR3を呼び出す。

EXPR3の結果の負号を反転し、XP24(加算処理)へジャンプする。
```
XP21   RST  1         ;POSITIVE SIGN?  IGNORE
       DB   '+' 
       DB   0Q
XP22   CALL EXPR3     ;1ST <EXPR3> 
```

正の符号で始まれば、単純に無視する。

EXPR3を呼び出し、第一項の値を計算する。
```
XP23   RST  1         ;ADD?
       DB   '+' 
       DB   25Q 
       PUSH H         ;YES, SAVE VALUE 
       CALL EXPR3     ;GET 2ND<EXPR3> 
```

加算であれば、第一項をスタックに保存し、EXPR3を呼び出して第二項の計算を行う。

加算でなければ、XP25へジャンプし、減算かどうかチェックする。
```
XP24   XCHG           ;2ND IN DE 
       XTHL           ;1ST IN HL 
       MOV  A,H       ;COMPARE SIGN
       XRA  D 
       MOV  A,D 
       DAD  D 
       POP  D         ;RESTORE TEXT POINTER
       JM   XP23      ;1ST 2ND SIGN DIFFER 
       XRA  H         ;1ST 2ND SIGN EQUAL
       JP   XP23      ;SO ISp RESULT
       JMP  QHOW      ;ELSE WE HAVE OVERFLOW 
```

HLの内容(第二項)をDEレジスタに入れる。DEレジスタの内容(テキストポインタ)はHLレジスタに入る。

HLレジスタへスタックポインタの最上位データ(第一項)を読み出し、代わりにスタックへテキストポインタの内容を保管する。

HLとDEの符号を比較し、サインフラグを変化させる。

HL=HL+DEとし、DEレジスタへテキストポインタを復元する。

第一項と第二項の符号が異なっていれば、オーバーフローは生じない。XP23へジャンプし(JM XP23)、次の演算を処理する。

第一項と第二項は符号が同じであれば、演算結果の上位バイトとDレジスタ(第二項の上位バイト)を比較する。これらの符号が同じであれば、演算前後で符号は変わっていない。オーバーフローは発生していないと判断できるので、XP23へジャンプし、次の演算を行う。符号が違えばオーバーフローが発生しているため、エラー処理を行う。
```
XP25   RST  1         ;SUBTRACT? 
       DB   '-' 
       DB   203Q
```

減算であれば、後続のEXPR3を実行する。

減算でなければXP42へジャンプし、呼び出し元へ戻る。
```
EXPR3  CALL EXPR4     ;GET 1ST <EXPR4> 
XP31   RST  1         ;MULTIPLY? 
       DB   '*' 
       DB   54Q 
```

EXPR4を呼び出し、第一項を計算する。

乗算記号が続けば、後続の処理で乗算を行う。乗算記号でなければ、除算のチェックを行うためにXP34へジャンプする。
```
       PUSH H         ;YES, SAVE 1ST 
       CALL EXPR4     ;AND GET 2ND <EXPR4> 
       MVI  B,0Q      ;CLEAR B FOR SIGN
       CALL CHKSGN    ;CHECK SIGN
       XCHG           ;2ND IN DE NOW 
       XTHL           ;1ST IN HL 
       CALL CHKSGN    ;CHECK SIGN OF 1ST 
```

乗算記号であれば、第一項をスタックへ退避したあと、再度EXPR4を呼び出し、第二項を計算する。

各項の符号チェックを行う。

CHKSGNを2回呼んでいるので、Bレジスタの値は第一項と第二項の符号が同じ場合は0、違えば1になる。
```
       MOV  A,H       ;IS HL > 255 ? 
       ORA  A 
       JZ   XP32      ;NO
       MOV  A,D       ;YES, HOW ABOUT DE 
       ORA  D 
       XCHG           ;PUT SMALLER IN HL 
       JNZ  AHOW      ;ALSO >, WILL OVERFLOW 
XP32   MOV  A,L       ;THIS IS DUMB
       LXI  H,0Q      ;CLEAR RESULT
       ORA  A         ;ADD AND COUNT 
       JZ   XP35
XP33   DAD  D 
       JC   AHOW      ;OVERFLOW
       DCR  A 
       JNZ  XP33
       JMP  XP35      ;FINISHED
```

HLまたはDEのいずれかが255以下である必要がある(両方が255より大きければ、オーバーフローが生じるため)。この条件を満たさない場合はエラーとする(AHOW)。

XP32に到達したとき、HL<=255となっている。AにLを代入し、A=0ならXP35にジャンプし乗算処理を終了する。

XP33では、HL=HL+DEとして、Aをデクリメントする。A=0となるまでXP33の処理を繰り返し、A=0となったら、XP35へジャンプしてEXPR3の終了処理を行う。
```
XP34   RST  1         ;DIVIDE? 
       DB   '/' 
       DB   104Q
```

除算記号が続くかどうか確認する。除算記号であれば処理を継続、除算記号でなければ、XP42へジャンプし、呼び出し元へ戻る。
```
       PUSH H         ;YES, SAVE 1ST <EXPR4> 
       CALL EXPR4     ;AND GET 2ND ONE 
       MVI  B,0Q      ;CLEAR B FOR SIGN
       CALL CHKSGN    ;CHECK SIGN OF 2ND 
       XCHG           ;PUT 2ND IN DE 
       XTHL           ;GET 1ST IN HL 
       CALL CHKSGN    ;CHECK SIGN OF 1ST 
```

乗算と同様に第一項をスタックへ退避し、第二項をEXPR4を呼び出すことで評価する。各項の符号チェックを行うところまで同じ。
```
       MOV  A,D       ;DIVIDE BY 0?
       ORA  E 
       JZ   AHOW      ;SAY "HOW?"
       PUSH B         ;ELSE SAVE SIGN
       CALL DIVIDE    ;USE SUBROUTINE
       MOV  H,B       ;RESULT IN HL NOW
       MOV  L,C 
       POP  B         ;GET SIGN BACK 
```

第二項が0でないかどうかを確認する。0であれば0除算のエラーになる。

符号を含むBCレジスタをスタックに保存し、除算ルーチン(DIVIDE)を呼び出す。

DIVIDEは結果がBCレジスタに入るので、HLレジスタへ結果を入れる。

スタックから符号をBCレジスタに復元する。
```
XP35   POP  D         ;AND TEXT POINTER
       MOV  A,H       ;HL MUST BE +
       ORA  A 
       JM   QHOW      ;ELSE IT IS OVERFLOW 
       MOV  A,B 
       ORA  A 
       CM   CHGSGN    ;CHANGE SIGN IFF NEEDED 
       JMP  XP31      ;LOOK OR MORE TERMS 
```

XP35は乗算、除算共通の終了処理になる。

スタックからテキストポインタをDEレジスタに復元する。

HLレジスタ(乗算結果、または除算結果)は正の値を取っているはずである。そうでなければオーバーフローであるとし、エラー処理を行う(QHOW)。

Bレジスタが示す符号が負であれば、CHGSGNを呼び出してHLの符号を変更する。

XP31へジャンプし、後続の項の演算を行う。
```
EXPR4  LXI  H,TAB4-1  ;FIND FUNCTION IN TAB4 
       JMP  EXEC      ;AND GO DO IT
```

EXPR4はTAB4をコマンドテーブルに設定し、関数を探す。関数が見つからなければ、XP40を実行する。
```
XP40   RST  7         ;NO, NOT A FUNCTION
       JC   XP41      ;NOR A VARIABLE
       MOV  A,M       ;VARIABLE
       INX  H 
       MOV  H,M       ;VALUE IN HL 
       MOV  L,A 
       RET
```

関数が見つからなかった場合、TSTV(RST 7)を呼び出し、変数であるかどうかを確認する。

変数だった場合、HLレジスタに値を読み込み、呼び出し元に戻る。
```
XP41   CALL TSTNUM    ;OR IS IT A NUMBER 
       MOV  A,B       ;# OF DIGIT
       ORA  A 
       RNZ            ;OK
```

変数でもなかった場合、数値かどうかを確認する。数値であればHLレジスタに値が入るので、呼び出し元に戻る。
```
PARN   RST  1         ;NO DIGIT, MUST BE 
       DB   '(' 
       DB   5Q
       RST  3         ;"(EXPR)"
       RST  1 
       DB   ')' 
       DB   1Q
XP42   RET
XP43   JMP  QWHAT     ;ELSE SAY: "WHAT?" 
```

数値でもなかった場合、括弧かどうかを確認する。

括弧であれば、EXPRを呼び出し、括弧の中を評価する。

括弧でなければエラーとする。また、EXPRのあとが閉じ括弧でなければ、これもエラーとする。

## 内部処理
### SETVAL

SETVALは変数に値を設定する。
```
SETVAL RST  7         ;*** SETVAL ***
       JC   QWHAT     ;"WHAT?" NO VARIABLE 
       PUSH H         ;SAVE ADDRESS OF VAR.
       RST  1         ;PASS "=" SIGN 
       DB   '=' 
       DB   10Q 
       RST  3         ;EVALUATE EXPR.
       MOV  B,H       ;VALUE IN BC NOW 
       MOV  C,L 
       POP  H         ;GET ADDRESS 
       MOV  M,C       ;SAVE VALUE
       INX  H 
       MOV  M,B 
       RET
SV1    JMP  QWHAT     ;NO "=" SIGN 
```

最初にTSTV(RST 7)を呼び出して、テキストポインタが指している文字が変数であるかどうかを確認する。変数であればアドレス(HLレジスタ)をスタックに保管する。

次にTSTC(RST 1)を呼び出し、'='が続くかどうか確認する。'='でなければSV1へジャンプし、エラー処理をする。

'='であれば、EXPR(RST 3)を呼び出して、右辺を評価する。

右辺の値をBCレジスタに移動し、スタックから変数のアドレスをHLレジスタに復元する。

HLレジスタが指すアドレスにBCレジスタの値を入れる。
### IGNBLK(RST 5)

空白を読み飛ばす。
```
SS1:   EQU  28H       ;EXECUTE TIME LOCATION OF THIS INSTRUCTION.
       LDAX D         ;*** IGNBLK/RST 5 ***
       CPI  40Q       ;IGNORE BLANKS 
       RNZ            ;IN TEXT (WHERE DE->)
       INX  D         ;AND RETURN THE FIRST
       JMP  SS1       ;NON-BLANK CHAR. IN A
```

テキストポインタ(DEレジスタ)が指すメモリの内容を' 'と比較する。

一致すれば呼び出し元に戻る。

一致しなければ、テキストポインタをインクリメントし、SS1に戻る。
### TSTC(RST 1)

TSTCを呼び出す際に使用される'RST 1'命令の後ろに続く命令は、例えば次のようになっている。
```
PR0    RST  1         ;ELSE IS IT FORMAT?
       DB   '#' 
       DB   5Q
       RST  3         ;YES, EVALUATE EXPR. 
```

'RST 1'命令は機械語サブルーチンへ移動する際に、スタックに自分自身の次のアドレスをプッシュする。通常はこのアドレスはサブルーチンの戻りアドレスとなるが、この場合は戻りアドレスに有効な命令が書いてあるわけではなく、'#'が書いてある。
```
       XTHL           ;*** TSTC OR RST 1 *** 
```

最初のXTHL命令でスタックに積んであるアドレス('#'を指している)をHLレジスタへ読み込む。
```
       RST  5         ;IGNORE BLANKS AND 
       CMP  M         ;TEST CHARACTER
       JMP  TC1       ;REST OF THIS IS AT TC1
```

空白を読み飛ばし、次の文字をHLレジスタが指すメモリの内容と比較する。

'RST 1'命令で呼び出せる機械語プログラム領域は8バイトしかない。続きのプログラムが別の場所(TC1)に書いてあるので、JMP命令で移動する。
```
TC1    INX  H         ;COMPARE THE BYTE THAT 
       JZ   TC2       ;FOLLOWS THE RST INST. 
       PUSH B         ;WITH THE TEXT (DE->)
       MOV  C,M       ;IFF NOT =, ADD THE 2ND 
       MVI  B,0       ;BYTE THAT FOLLOWS THE 
       DAD  B         ;RST TO THE OLD PC 
       POP  B         ;I.E., DO A RELATIVE 
       DCX  D         ;JUMP IFF NOT = 
TC2    INX  D         ;IFF =, SKIP THOSE BYTES
       INX  H         ;AND CONTINUE
       XTHL 
       RET
```

先の比較で、HLレジスタが指すメモリの内容とテキストポインタが指す内容が一致すれば、TC2へ移動し、テキストポインタを一つ進め、HLアドレスは合計2つすすめる。この時点でHLレジスタは'#'と'5Q'を読み飛ばし、'RST 3'を指している。

'XTHL'命令でHLレジスタの内容をスタックの一番上に積まれた内容と交換し、次のRET命令で'RST 3'にジャンプする。

HLレジスタが指すメモリの内容と、テキストポインタが指す内容が一致しなければ、HLレジスタを一つ進め、BCレジスタにHLレジスタが指す内容('5Q')を読み込み、HLレジスタに加算する。

TC2へ移動したあと、テキストポインタがインクリメントされるので、予めテキストポインタをデクリメントしておき、TSTCを抜けたとき、テキストポインタが空白を読み飛ばした直後の位置になるようにしておく。

最後に加算後のHLレジスタをスタックに戻し、RET命令でHLレジスタが指す位置へジャンプする。
### TSTV(RST 7)

TSTV(RST 7(はテキストポンタが指している文字が変数であるかどうかを確認する。
```
       RST  5         ;*** TSTV OR RST 7 *** 
       SUI  100Q      ;TEST VARIABLES
       RC             ;C:NOT A VARIABLE
       JMP  TSTV1     ;JUMP AROUND RESERVED AREA
```

最初にテキストポインタが指す文字と'@(100Q)'を比較する。'@'よりも小さければ、変数ではないので呼び出し元へ戻る。

'@'以上であれば、TSTV1へジャンプする。
```
TSTV1  JNZ  TV1       ;NOT "@" ARRAY 
       INX  D         ;IT IS THE "@" ARRAY 
       CALL PARN      ;@ SHOULD BE FOLLOWED
       DAD  H         ;BY (EXPR) AS ITS INDEX
       JC   QHOW      ;IS INDEX TOO BIG? 
       PUSH D         ;WILL IT OVERWRITE 
       XCHG           ;TEXT? 
       CALL SIZE      ;FIND SIZE OF FREE 
       RST  4         ;AND CHECK THAT
       JC   ASORRY    ;IFF SO, SAY "SORRY"
```

ます、テキストポインタが指す文字が'@'に等しいかどうかを確認する。等しければ配列なので、処理を継続、等しくなければ通常の変数なので、TV1以降で処理する。

PARNで'@'に続く括弧の中の式を評価する。括弧が続いてなければ、PARN内でエラー処理を行う。

式の評価結果は配列のインデックスになる。データサイズは16ビットなので、評価結果を２倍する(DAD H)。

インデックスが大きすぎないかチェックを行う。大きすぎればエラー処理(ASORRY)を行う。
```
SS1A   LXI  H,VARBGN  ;IFF NOT, GET ADDRESS 
       CALL SUBDE     ;OF @(EXPR) AND PUT IT 
       POP  D         ;IN HL 
       RET            ;C FLAG IS CLEARED 
```

配列の要素はVARBGNからINDEXだけ下位のバイトが格納位置になる。配列変数の場合はここで呼び出し元に戻る。結果はHLレジスタに入っている。
```
TV1    CPI  33Q       ;NOT @, IS IT A TO Z?
       CMC            ;IFF NOT RETURN C FLAG
       RC 
       INX  D         ;IFF A THROUGH Z
TV1A   LXI  H,VARBGN  ;COMPUTE ADDRESS OF
       RLC            ;THAT VARIABLE 
       ADD  L         ;AND RETURN IT IN HL 
       MOV  L,A       ;WITH C FLAG CLEARED 
       MVI  A,0 
       ADC  H 
       MOV  H,A 
       RET
```


### QTSTG

QTSTGは引用符がついた文字列を印字する。また、"_"であった場合、RI(Reverse Line Feed)を印字する。
```
QTSTG  RST  1         ;*** QTSTG *** 
       DB   '"' 
       DB   17Q 
```

テキストポインタが二重引用符を指しているかどうか確認する。
```
       MVI  A,42Q     ;IT IS A " 
QT1    CALL PRTSTG    ;PRINT UNTIL ANOTHER 
       CPI  0DH       ;WAS LAST ONE A CR?
       POP  H         ;RETURN ADDRESS
       JZ   RUNNXL    ;WAS CR, RUN NEXT LINE 
```

二重引用符を指していれば、次の二重引用符まで印字する。

印字した文字列の次が改行コードであれば、スタック上の戻りアドレスを廃棄し、次の行を実行する。
```
QT2    INX  H         ;SKIP 3 BYTES ON RETURN
       INX  H 
       INX  H 
       PCHL           ;RETURN
```

改行コードでなければ、呼び出し元に戻るが、CALL QTSTGのあとはJMP命令が続いている。QTSTG呼び出し時のテキストポインタは文字列を指していたので、これをスキップするために戻りアドレスを+3してから戻る。
```
QT3    RST  1         ;IS IT A ' ? 
       DB   47Q 
       DB   5Q
       MVI  A,47Q     ;YES, DO SAME
       JMP  QT1       ;AS IN " 
```

二重引用符ではなかった場合、単一引用符かどうかを調べて、単一引用符のときと同じ処理をする。
```
QT4    RST  1         ;IS IT BACK-ARROW? 
       DB   137Q
       DB   10Q 
       MVI  A,215Q    ;YES, 0DHWITHOUT LF!!
       RST  2         ;DO IT TWICE TO GIVE 
       RST  2         ;TTY ENOUGH TIME 
       POP  H         ;RETURN ADDRESS
       JMP  QT2 
QT5    RET            ;NONE OF ABOVE 
```

単一引用符でもなかった場合、"_"かどうか確認する。

"_"だった場合、RI(Reverse Line Feed)を二度送る。この場合も戻りアドレスを+3してからQTSTGから戻る。

"_"ではなかった場合、QT5へジャンプし、呼び出し元に戻る。この場合、文字列ではなかったので"CALL QTSTG"のあとのJMP命令を実行する。
## 実行制御
### FINISH(RST 6)

FINISH(RST 6)は文の終わりで呼び出される。
```
       POP  PSW       ;*** FINISH/RST 6 ***
       CALL FIN       ;CHECK END OF COMMAND
       JMP  QWHAT     ;PRINT "WHAT?" IFF WRONG
```

FINISH(RST 6)は呼び出し元へはRETで戻らないので、'POP PSW'でスタックに積まれた戻りアドレスを読み捨てる。

次に処理の本体(FIN)を呼び出す。

FINはエラー時のみRETで戻ってくる。エラーで戻ってきたら、QWHATへ飛び、メインループに戻る。
```
FIN    RST  1         ;*** FIN *** 
       DB   73Q 
       DB   4Q 
       POP  PSW       ;";", PURGE RET ADDR.
       JMP  RUNSML    ;CONTINUE SAME LINE
```

まず、次の文字が';'かどうかをチェックする。

';'であれば、戻りアドレスを読み捨て、RUNSMLに飛んで後続の文を実行する。

';'でなければFI1へジャンプする。
```
FI1    RST  1         ;NOT ";", IS IT CR?
       DB   0DH
       DB   4Q 
       POP  PSW       ;YES, PURGE RET ADDR.
       JMP  RUNNXL    ;RUN NEXT LINE 
FI2    RET            ;ELSE RETURN TO CALLER 
```

次に、';'でなかった場合、'\r'かどうか確認する。

'\r'であれば、戻りアドレスを読み捨て、RUNNXLに飛んで次の行を実行する。

'\r'でなければ、呼び出し元(FINISH)に戻り、エラー処理を行う。
### FNDLN

FNDLNはHLレジスタに入った行番号をもつ行を検索する。
```
FNDLN  MOV  A,H       ;*** FNDLN *** 
       ORA  A         ;CHECK SIGN OF HL
       JM   QHOW      ;IT CANNT BE -
       LXI  D,TXTBGN  ;INIT. TEXT POINTER
```

行番号は負であってはならない。負であればエラーとする(QHOW)。

エラーでなければ、テキストポインタ(DEレジスタ)にプログラムバッファの先頭アドレスをセットする。
```
FNDLNP EQU  $         ;*** FNDLNP ***
FL1    PUSH H         ;SAVE LINE # 
       LHLD TXTUNF    ;CHECK IFF WE PASSED END
       DCX  H 
       RST  4 
       POP  H         ;GET LINE # BACK 
       RC             ;C,NZ PASSED END 
       LDAX D         ;WE DID NOT, GET BYTE 1
       SUB  L         ;IS THIS THE LINE? 
       MOV  B,A       ;COMPARE LOW ORDER 
       INX  D 
       LDAX D         ;GET BYTE 2
       SBB  H         ;COMPARE HIGH ORDER
       JC   FL2       ;NO, NOT THERE YET 
       DCX  D         ;ELSE WE EITHER FOUND
       ORA  B         ;IT, OR IT IS NOT THERE
       RET            ;NC,Z:FOUND; NC,NZ:NO
```

FNDLNPは、行番号(HL)をもつ行をテキストポインタがさすアドレス以降の領域から検索する。

まず、テキストポインタがプログラムテキスト領域の終了位置より後ろにないことをCOMP(RST 4)を呼び出して確認する。後ろにあれば、呼び出し元に戻る。

テキストポインタが指す行の行番号(2バイト)を読み出し、探している行番号(HLレジスタ)と比較する。

テキストポインタが指す行番号が、探している行番号より小さければ、FL2に飛んで行末まで読み飛ばし(FNDSKP)、検索を続ける。

テキストポインタが指す行番号が、探している行と同じであるか、探している行番号より大きくなれば、呼び出し元に戻る。行が見つかったか見つからなかったかはフラグレジスタの内容で判断する。
```
FNDNXT EQU  $         ;*** FNDNXT ***
       INX  D         ;FIND NEXT LINE
FL2    INX  D         ;JUST PASSED BYTE 1 & 2
;* 
FNDSKP LDAX D         ;*** FNDSKP ***
       CPI  0DH       ;TRY TO FIND 0DH
       JNZ  FL2       ;KEEP LOOKING
       INX  D         ;FOUND CR, SKIP OVER 
       JMP  FL1       ;CHECK IFF END OF TEXT
```

FNDNXTは次の行の先頭から検索を行う。

FNDNXTがCALLされたときは、行番号の2バイトを読み飛ばす。

FNDLNPからジャンプしてきたときは、テキストポインタは行番号の2バイト目を指しているので、1バイトだけ読み飛ばす。

FNDSKPは'\r'が見つかるまでテキストポインタをインクリメントする。

'\r'が見つかったら、テキストポインタをもう一回インクリメントし、次の行の行番号を指すようにしてからFL1へジャンプして検索を始める。
### PUSHA
```
PUSHA  LXI  H,STKLMT  ;*** PUSHA *** 
       CALL CHGSGN
       POP  B         ;BC=RETURN ADDRESS 
       DAD  SP        ;IS STACK NEAR THE TOP?
       JNC  QSORRY    ;YES, SORRY FOR THAT.
```

まず、スタックの上限に近づいていないか確認する。

スタックの上限に近づいていれば、エラー処理を行う。

ループ変数をスタックに積む前に、戻りアドレスをBCレジスタに読み込んでおく。
```
       LHLD LOPVAR    ;ELSE SAVE LOOP VAR.S
       MOV  A,H       ;BUT IFF LOPVAR IS 0
       ORA  L         ;THAT WILL BE ALL
       JZ   PU1 
```

ループ変数(LOPVAR)をチェックする。LOPVAR=0であれば、LOPVAR以外のFOR変数はスタックに積まずにPU1にジャンプする。
```
       LHLD LOPPT     ;ELSE, MORE TO SAVE
       PUSH H 
       LHLD LOPLN 
       PUSH H 
       LHLD LOPLMT
       PUSH H 
       LHLD LOPINC
       PUSH H 
       LHLD LOPVAR
```

LOPVARが0でなければ、ループ変数をすべてスタックに積む。
```
PU1    PUSH H 
       PUSH B         ;BC = RETURN ADDR. 
       RET

最後にLOPVARと戻りアドレスをスタックにプッシュする。
```
### POPA
```
POPA   POP  B         ;BC = RETURN ADDR. 
       POP  H         ;RESTORE LOPVAR, BUT 
       SHLD LOPVAR    ;=0 MEANS NO MORE
       MOV  A,H 
       ORA  L 
       JZ   PP1       ;YEP, GO RETURN
```

まず戻りアドレスをBCレジスタに退避し、LOPVARを取り出す。

LOPVAR=0であれば、もうFOR変数は積まれていないので、PP1にジャンプする。
```
       POP  H         ;NOP, RESTORE OTHERS 
       SHLD LOPINC
       POP  H 
       SHLD LOPLMT
       POP  H 
       SHLD LOPLN 
       POP  H 
       SHLD LOPPT 
```

FOR変数をスタックから取り出す。
```
PP1    PUSH B         ;BC = RETURN ADDR. 
       RET
```

戻りアドレスをスタックに戻し、RETで呼び出し元に戻る。
## バッファ操作
### MVUP

MVUPはDEからHL-1までのメモリブロックをBCレジスタが指すアドレスから始まるメモリブロックに移動する。

BC<DEである必要がある。

| ADR  |                 |
|------|-----------------|
| BC   | BEGIN(NEW)      |
|      | END(NEW)        |
|------|-----------------|
|      |                 |
|------|-----------------|
| DE   | BEGIN(ORIGINAL) |
| HL-1 | END(ORIGINAL)   |

```
MVUP   RST  4         ;*** MVUP ***
       RZ             ;DE = HL, RETURN 
       LDAX D         ;GET ONE BYTE
       STAX B         ;MOVE IT 
       INX  D         ;INCREASE BOTH POINTERS
       INX  B 
       JMP  MVUP      ;UNTIL DONE
```

HL(移動先)とDE(移動元)のアドレスを比較する。HL=DEなら終了する。

DEが指すメモリの内容を読み取り、BCレジスタが指すメモリにコピーする。

DEとBCをインクリメントし、MVUPに戻る。データのコピーをHL=DEとなるまで繰り返す。
### MVDOWN

MVDOWNはBC〜DE-1で表されるメモリブロックを、HLレジスタが指すアドレス-1で終わるメモリブロックに移動する。

BC>DEである必要がある。

| ADR  |                 |
|------|-----------------|
| BC   | BEGIN(ORIGINAL) |
| DE-1 | END(ORIGINAL)   |
|------|-----------------|
|      |                 |
|------|-----------------|
|      | BEGIN(NEW)      |
| HL-1 | END(NEW)        |

```
MVDOWN MOV  A,B       ;*** MVDOWN ***
       SUB  D         ;TEST IFF DE = BC 
       JNZ  MD1       ;NO, GO MOVE 
       MOV  A,C       ;MAYBE, OTHER BYTE?
       SUB  E 
       RZ             ;YES, RETURN 
MD1    DCX  D         ;ELSE MOVE A BYTE
       DCX  H         ;BUT FIRST DECREASE
       LDAX D         ;BOTH POINTERS AND 
       MOV  M,A       ;THEN DO IT
       JMP  MVDOWN    ;LOOP BACK 
```

BC=DEであれば、処理を終了する。そうでなければMD1へ進む。

先にDEとHLをデクリメントする。

DEが指すメモリの内容をHLレジスタが指すメモリにコピーする。

MVDOWNへジャンプし、BC=DEとなるまで処理を繰り返す。
## エラー処理
### ERROR

エラーメッセージには"WHAT?", "SORRY?", "HOW?"がある。

エラーメッセージを表示したら、RSTARTへ戻るか、入力エラーの場合は再入力を促す。
```
QHOW   PUSH D         ;*** ERROR: "HOW?" *** 
AHOW   LXI  D,HOW 
       JMP  ERROR 
```

QHOWとAHOWの違いは、テキストポインタを保存するかどうかである。呼び出し時にスタックにテキストポインタを退避済みであれば、AHOW、まだ退避していなければQHOWを呼び出す。

QHOWはテキストポインタ(DEレジスタ)をスタックに退避したあと、DEレジスタに"HOW"文字列を設定し、ERRORルーチンへジャンプする。
```
QSORRY PUSH D         ;*** QSORRY ***
ASORRY LXI  D,SORRY   ;*** ASORRY ***
       JMP  ERROR 
```

QSORRYとASORRY、QWHATとAWHATも同様であるが、QWHATはERRORルーチンの前に配置されているため、JMP命令は使用しない。
```
QWHAT  PUSH D         ;*** QWHAT *** 
AWHAT  LXI  D,WHAT    ;*** AWHAT *** 
ERROR  SUB  A         ;*** ERROR *** 
       CALL PRTSTG    ;PRINT 'WHAT?', 'HOW?' 
       POP  D         ;OR 'SORRY'
       LDAX D         ;SAVE THE CHARACTER
       PUSH PSW       ;AT WHERE OLD DE ->
       SUB  A         ;AND PUT A 0 THERE 
       STAX D 
       LHLD CURRNT    ;GET CURRENT LINE #
       PUSH H 
       MOV  A,M       ;CHECK THE VALUE 
       INX  H 
       ORA  M 
       POP  D 
       JZ   RSTART    ;IFF ZERO, JUST RERSTART
       MOV  A,M       ;IFF NEGATIVE,
       ORA  A 
       JM   INPERR    ;REDO INPUT
       CALL PRTLN     ;ELSE PRINT THE LINE 
       DCX  D         ;UPTO WHERE THE 0 IS 
       POP  PSW       ;RESTORE THE CHARACTER 
       STAX D 
       MVI  A,77Q     ;PRINTt A "?" 
       RST  2 
       SUB  A         ;AND THE REST OF THE 
       CALL PRTSTG    ;LINE
       JMP  RSTART
```

テキストポインタにエラー文字列を設定した後、Aレジスタをクリアし、PRTSTGを呼び出すことでエラー文字列を印字する。

テキストポインタを復元し、テキストポインタが指す最初の文字をスタックに退避しておいてから、その文字を'0'に置き換える。

現在の行番号を取得し、行番号が0(ダイレクト実行を表す)であれば、RSTARTへジャンプし、リスタートする。

行番号が負であれば、INPUT文での入力エラーであると判断し、再入力を促す。

行番号が正であれば、現在行を先頭から'0'で置き換えた箇所まで印字する(PRTLNは'0'を見つけると印字を終了する)。

'0'で置き換えた箇所まで印字した後、'0'を元の文字に戻し、'?'を印字する。

その後、現在行の残りを印字する。
```
HOW    DB   'HOW?',0DH 
OK     DB   'OK',0DH 
WHAT   DB   'WHAT?',0DH 
SORRY  DB   'SORRY',0DH 
```

## 関数

関数テーブル(TAB4)に登録されているTiny Basic関数の処理を行うサブルーチン群の説明を行う。
### ABS

ABS関数の引数の絶対値を求める。
```
ABS    CALL PARN      ;*** ABS(EXPR) *** 
       CALL CHKSGN    ;CHECK SIGN
       MOV  A,H       ;NOTE THAT -32768
       ORA  H         ;CANNOT CHANGE SIGN
       JM   QHOW      ;SO SAY: "HOW?"
       RET
```

引数を評価し、その結果の符号をCHKSGNで確認する。

CHKSGNはHLレジスタの値が負であれば2の補数を求めることで絶対値を求めるが、HLレジスタの値が0x8000の場合、2の補数も0x8000であるため、絶対値を求めることができない。この場合、エラー処理(QHOW)を行う。
### SIZE

プログラムテキスト領域の残りバイト数を求める。
```
SIZE   LHLD TXTUNF    ;*** SIZE ***
       PUSH D         ;GET THE NUMBER OF FREE
       XCHG           ;BYTES BETWEEN 'TXTUNF'
SIZEA  LXI  H,VARBGN  ;AND 'VARBGN'
       CALL SUBDE 
       POP  D 
       RET
```

VARBGN(変数領域の先頭)からTXTUNF(プログラムテキストの終了位置)を減算することでプログラムテキスト領域の残りバイト数を求める。
### RND

RND(I)は乱数を計算する。
```
RND    CALL PARN      ;*** RND(EXPR) *** 
       MOV  A,H       ;EXPR MUST BE +
       ORA  A 
       JM   QHOW
       ORA  L         ;AND NON-ZERO
       JZ   QHOW
       PUSH D         ;SAVE BOTH 
       PUSH H 
```

まず、PARNを呼び出し、引数を評価する。引数は正である必要がある。正でなければエラー処理(QHOW)を行う。

テキストポインタ(DEレジスタ)と引数(HLレジスタ)をスタックに退避する。
```
       LHLD RANPNT    ;GET MEMORY AS RANDOM
       LXI  D,LSTROM  ;NUMBER
       RST  4 
       JC   RA1       ;WRAP AROUND IFF LAST 
       LXI  H,START 
```

HLレジスタにRANPNTの値(初期値はSTART)を読み込み、LSTROMと同じかどうか確認する。

LSTROMと同じであれば、HLレジスタの値を初期値(START)にリセットする。
```
RA1    MOV  E,M 
       INX  H 
       MOV  D,M 
       SHLD RANPNT
       POP  H 
       XCHG 
       PUSH B 
       CALL DIVIDE    ;RND(N)=MOD(M,N)+1 
       POP  B 
       POP  D 
       INX  H 
       RET
```

HLレジスタが指すメモリの値をDEレジスタに読み込む。この過程でHLレジスタの値は+1される。

HLレジスタの値をRANPNTへ保存する。

引数をスタックからHLレジスタに読み出す。

BCレジスタをスタックへ退避してから、DIVIDEを呼び出し、HL/DEを計算する。DIVIDEはHLレジスタに剰余を保管して戻る。

BCレジスタとDEレジスタ(テキストポインタ)をスタックから復元し、HLに1を加える。

まとめると、この関数は引数をメモリの内容で除算し、その剰余に1を加えたものを乱数としている。除数となるメモリのアドレスはRNDを呼び出すたびに+1される。
### INP

INP(I)はポート'I'から値を読み、その値を返す。
```
INP    CALL PARN
       MOV  A,L
       STA  INPIO + 1
       MVI  H,0
       JMP  INPIO
       JMP  QWHAT
```

PARNを使って引数を評価する。結果の下位バイト(Lレジスタ)をINPIO+1に書き込む。
```
INPIO  IN   0FFH
       MOV  L,A
       RET
```

INPIO+1はIN命令の引数になっている。ソースコード上は0FFHになっているが、ここにジャンプしてくる前にINP(I)命令の引数に書き換えられている。

IN命令を実行後、結果(Aレジスタ)をLレジスタにコピーし、呼び出し元に戻る。
### PEEK

PEEK(I)は引数Iで表されるアドレスの値を返す。
```
PEEK   CALL PARN
       MOV  L,M
       MVI  H,0
       RET
```

PARNを呼び出し、引数を評価する。

評価結果(HLレジスタ)が指すメモリの値をLレジスタに入れる。

PEEKの戻り値は8ビットなので、Hレジスタには0を入れて呼び出し元に戻る。
### POKE

POKE I, Jはアドレス'I'にデータ'J'を書き込む。アドレスとデータの組は一つである必要はなく、POKE I, J, K, Lの様に書くことでアドレス'K'にデータ'L'を書き込むことができる。引数には括弧をつけてはならない。
```
POKE   RST  3
       PUSH H
       RST  1
       DB   ','
       DB   12H
       RST  3
       MOV  A,L
       POP  H
       MOV  M,A
       RST  1
       DB   ',',03H
       JMP  POKE
       RST 6
```

EXPRを呼び出し、第一引数を評価する。評価結果(HLレジスタ)をスタックに保存する。

TSTC(RST 1)を使って、次の文字が','であることを確認する。','でなければFINISH(RST 6)を呼び出す。

','であれば、第二引数を評価する。評価結果の下位バイト(Lレジスタ)をAレジスタに入れる。

スタックから第一引数を読み出し、第二引数の値を第一引数で示されるメモリに書き込む。

TSTC(RST 1)を再び呼び出し、次の文字を確認する。次の文字が','であれば、第三引数があると判断し、POKEへジャンプする。

','でなければFINISH(RST 6)を呼び出す。
### USR

USR(I(, J))はアドレス'I'に配置されている機械語ルーチンを実行する。引数'J'が与えられている場合は、'J'の値をHLレジスタに設定して機械語ルーチンを呼び出す。
```
USR    PUSH B
       RST  1
       DB   '(',28D    ;QWHAT
       RST  3          ;EXPR
       RST  1
       DB   ')',7      ;PASPARM
       PUSH D
       LXI  D,USRET
       PUSH D
       PUSH H
       RET             ;CALL USR ROUTINE
```
```
USRET  POP  D
       POP  B
       RET
       JMP  QWHAT
```

BCレジスタをスタックに退避し、TSTC(RST 1)を呼び出して次の文字が'('であることを確認する。

EXPR(RST 3)を呼び出し、第一引数を評価する。HLレジスタにはユーザ定義の機械語ルーチンの先頭アドレスが入る。

TSTC(RST 1)を呼び出し、次の文字が')'であるかどうか確認する。')'でなければ、次の引数を評価するためにPASPRMへジャンプする。

次の文字が')'であった場合、テキストポインタ(DEレジスタ)をスタックに退避する。

次にユーザ定義の機械語ルーチンからの戻りアドレスとしてUSRETをスタックに積む。このようにしてユーザ定義の機械語ルーチンから戻ったときにBC、DEレジスタを復元する。

次に第一引数(HLレジスタ)をスタックにプッシュする。このあとRET命令を実行することで、ユーザ定義関数へジャンプする。
```
PASPRM RST  1
       DB   ',',14D
       PUSH H
       RST  3
       RST  1
       DB   ')',9
       POP  B
       PUSH D
       LXI  D,USRET
       PUSH D
       PUSH B
       RET             ;CALL USR ROUTINE
```

第二引数がある場合、PASPRMを実行する。

PASPRMでは最初に次の文字が','であるかどうかを確認する。','でなければUSRETへジャンプし、BCレジスタ、DEレジスタを復元して呼び出し元に戻る。

','であれば、第一引数(HLレジスタ)をスタックに退避したあと、EXPR(RST 3)を実行し、第二引数を評価する。

第二引数を評価したあと、次の文字が')'であるか確認する。')'であれば、スタックから第一引数を復元し、テキストポインタ、USRET、第一引数の順でスタックに積み、RET命令を実行する。RET命令を実行すると、スタックの一番上に積まれているデータ(第一引数)で示されるアドレスにジャンプする。HLレジスタには第二引数が保管されているので、ユーザ定義関数側ではHLの引数として使用することができる。

第二引数のあとの文字が')'でなければ、USRETへジャンプする。USRETでは第一引数をDEレジスタへ復元し、BCレジスタへはUSRを呼び出す前のBCレジスタの値を復元してから呼び出し元に戻る。テキストポインタの復元は、呼び出し元に戻ってから行う。
### XP40

テキストポインタが指している式が、関数テーブルに登録されていない場合の処理を行う。
```
XP40   RST  7         ;NO, NOT A FUNCTION
       JC   XP41      ;NOR A VARIABLE
       MOV  A,M       ;VARIABLE
       INX  H 
       MOV  H,M       ;VALUE IN HL 
       MOV  L,A 
       RET
XP41   CALL TSTNUM    ;OR IS IT A NUMBER 
       MOV  A,B       ;# OF DIGIT
       ORA  A 
       RNZ            ;OK
```

まずTSTV(RST 7)で変数であるかどうかを確認する。

変数でなければ、XP41へジャンプする。

変数であれば、変数の値をHLレジスタに設定し、呼び出し元に戻る。

XP41では、テキストポインタが数値文字列を指しているかどうかを確認する。

数値文字列であるかどうかは、桁数(Bレジスタ)を調べることで確認できる。

数値文字列であれば、呼び出し元に戻る。数値文字列でなければ、次のルーチン(PARN)を実行する。

## ユーティリティ
### DIVIDE(除算)

HLをDEで除算する。商をBC、剰余をHLに入れて戻る。

大まかなアルゴリズムは、HL<=0となるまでHLからDEを引き続け、引いた回数を商とする、というものである。
```
DIVIDE PUSH H         ;*** DIVIDE ***
       MOV  L,H       ;DIVIDE H BY DE
       MVI  H,0 
       CALL DV1 
       MOV  B,C       ;SAVE RESULT IN B
       MOV  A,L       ;(REMAINDER+L)/DE
       POP  H 
       MOV  H,A 
DV1    MVI  C,377Q    ;RESULT IN C 
DV2    INR  C         ;DUMB ROUTINE
       CALL SUBDE     ;DIVIDE BY SUBTRACT
       JNC  DV2       ;AND COUNT 
       DAD  D 
       RET
```

最初に上位8ビットだけを考える。HLの上位8ビットに対してDV1を呼び出す。

DV1はCに0xFFをセットするところから始まる。CはSUBDEを呼び出した回数をカウントするために使われている。C=0xFFとした直後にINRCでインクリメントするため、SUBDEを最初に呼び出すときにはC=0となっている。

SUBDEの結果が正であれば、DV2にもどり、減算を再実行する。

SUBDEの結果が正でなければ、DEを一回引きすぎているので、DAD DとしてHLにDEを加算してDV1の呼び出し元に戻る。

DV1の呼び出しから戻ると、減算回数をBレジスタに保存する。

(上位8ビット除算の余り*16+L)をHLレジスタに設定し、DV1の処理を行う。Cレジスタの内容は、Bレジスタと合わせて、HL/DEの商となる。今回のDV1の戻り先はDIVIDEを呼び出した箇所である。
### SUBDE(16ビット減算)

HL-DEを計算する。
```
SUBDE  MOV  A,L       ;*** SUBDE *** 
       SUB  E         ;SUBTRACT DE FROM
       MOV  L,A       ;HL
       MOV  A,H 
       SBB  D 
       MOV  H,A 
       RET
```

単純な16ビット減算である。

まず、L-Eを計算し、次にH-DをL-Eのボロー付きで計算する。
### CHKSGN(符号確認) / CHGSGN(符号変更)

HLレジスタの符号を確認し、負であれば符号を反転する。また、Bレジスタの最上位ビットも反転する。呼び出す前にBレジスタの最上位ビットをクリアしておけば、Bレジスタで符号、HLレジスタで絶対値を表現できる。
```
CHKSGN MOV  A,H       ;*** CHKSGN ***
       ORA  A         ;CHECK SIGN OF HL
       RP             ;IFF -, CHANGE SIGN 
;* 
```

H OR Hを計算して、Hレジスタの符号を確認する。符号が正であればそのまま呼び出し元に戻る。
```
CHGSGN MOV  A,H       ;*** CHGSGN ***
       CMA            ;CHANGE SIGN OF HL 
       MOV  H,A 
       MOV  A,L 
       CMA
       MOV  L,A 
       INX  H 
       MOV  A,B       ;AND ALSO FLIP B 
       XRI  200Q
       MOV  B,A 
       RET
```

HLレジスタの符号が負であれば、CHGSGNで符号を反転する。

2の補数を求めるために、HLの符号を反転し、1を加算する。

Bレジスタの符号も反転する。
### CKHLDE(16ビット比較)

16ビットデータの符号付き比較を行う。

符号が同じであれば、そのままCOMP(RST 4)で比較する。

符号が異なれば、HLとDEを入れ替えてからCOMP(RST 4)を呼び出す。
  - HL>0、DE<0の場合、期待される結果はHL>DEであるが、符号なしで比較するとHL<DEとなるため、期待した結果が得られない。
  - HL<0、DE>0の場合も同様で、期待される結果はHL<DEであるが、COMPが返す結果はHL>DEである。
  - このため、HLとDEの符号が違う場合、期待とは逆の結果になるため、COMP(RST 4)を呼び出す前にHLとDEを入れ替えておく。
```
CKHLDE MOV  A,H 
       XRA  D         ;SAME SIGN?
       JP   CK1       ;YES, COMPARE
       XCHG           ;NO, XCH AND COMP
CK1    RST  4 
       RET
```

### COMP

**16ビットデータの符号なし比較を行う。**

HL-DEを計算し、その結果のフラグを設定すれば良いが、8ビットずつ比較を実施する。
```
       MOV  A,H       ;*** COMP OR RST 4 *** 
       CMP  D         ;COMPARE HL WITH DE
       RNZ            ;RETURN CORRECT C AND
       MOV  A,L       ;Z FLAGS 
       CMP  E         ;BUT OLD A IS LOST 
       RET
```

H-Dを計算し、結果がゼロでなければ、呼び出し元に戻る。フラグはH-Dの結果がそのまま設定されている。

H=Dであれば、L-Eを計算し、結果がゼロでなければ呼び出し元に戻る。フラグにはL-Eの結果が設定される。

## 入出力
### GETLN

キーボードから一行読み込む
```
GETLN  RST  2         ;*** GETLN *** 
       LXI  D,BUFFER  ;PROMPT AND INIT
```

GETLNを呼び出す前にAレジスタにプロンプト文字を入れてある。最初にOUTC(RST 2)を呼び出してプロンプトを表示する。次にテキストポインタを行入力用のバッファを指すように設定する。
```
GL1    CALL CHKIO     ;CHECK KEYBOARD
       JZ   GL1       ;NO INPUT, WAIT
```

CHKIOを呼び出し、キーボードの状態を確認する。入力があるまでCHKIOを実行する。
```
       CPI  177Q      ;DELETE LST CHARACTER?
       JZ   GL3       ;YES 
       CPI  12Q       ;IGNORE LF 
       JZ   GL1 
       ORA  A         ;IGNORE NULL 
       JZ   GL1 
       CPI  134Q      ;DELETE THE WHOLE LINE?
       JZ   GL4       ;YES 
```

入力が7FHであった場合、GL3へジャンプする。

入力が0AH(改行コード)である場合、GL1へジャンプして再入力を行う。

入力が0である場合もGL1へジャンプして再入力を行う。

入力が5CHである場合はGL4へジャンプする。
```
       STAX D         ;ELSE, SAVE INPUT
       INX  D         ;AND BUMP POINTER
       CPI  15Q       ;WAS IT CR?
       JNZ  GL2       ;NO
       MVI  A,12Q     ;YES, GET LINE FEED
       RST  2         ;CALL OUTC AND LINE FEED
       RET            ;WE'VE GOT A LINE
```

入力(Aレジスタ)をテキストポインタ(DEレジスタ)が指すメモリに保存する。

テキストポインタ(DEレジスタ)を一つ進める。

入力が復帰コード(0DH)であれば、OUTC(RST 2)を呼び出して改行コード(0AH)を印字し、呼び出し元へ戻る。復帰コードでなければ、GL2へジャンプする。
```
GL2    MOV  A,E       ;MORE FREE ROOM?
       CPI  BUFEND AND 0FFH
       JNZ  GL1       ;YES, GET NEXT INPUT 
```

入力バッファに空きがあるか確認する。

テキストポインタの下位バイトを入力バッファの終わり(BUFEND)の下位バイトと比較する。入力バッファの大きさは80バイト(50H)なので、下位バイトの比較のみで十分である。一致しなければまだバッファに空きがあると判断し、GL1に戻って追加入力を受け付ける。一致した場合、GL3へ進む。
```
GL3    MOV  A,E       ;DELETE LAST CHARACTER 
       CPI  BUFFER AND 0FFH    ;BUT DO WE HAVE ANY? 
       JZ   GL4       ;NO, REDO WHOLE LINE 
       DCX  D         ;YES, BACKUP POINTER 
       MVI  A,'_'     ;AND ECHO A BACK-SPACE 
       RST  2 
       JMP  GL1       ;GO GET NEXT INPUT 
GL4    CALL CRLF      ;REDO ENTIRE LINE
       MVI  A,136Q    ;CR, LF AND UP-ARROW 
       JMP  GETLN 
```

GL3では最後の文字を一文字削除する。ここに来るのは、入力が5CHである場合、または入力バッファの最後に到達した場合である。

まず、入力バッファの先頭アドレス(BUFFER)とテキストポインタ(DEレジスタ)を比較する。前述した理由で比較は下位バイトのみで良い。一致しなければ、テキストポインタ(DEレジスタ)を一文字前へ戻し、バックスペース文字('_')を印字する。GGL1へ戻って追加入力を受け付ける。一致した場合は、テキストポインタを一つ前へ戻すことができない。GL4へジャンプする。

GL4では、復帰コード(0DH)と改行コード(0AH)、一行消去したことを意味する'^'を印字し、GETLNへ戻って最初から入力をやり直す。
### CHKIO

キーボードから一文字読み取る。
```
CHKIO  PUSH B         ;SAVE B ON STACK
       PUSH D         ;AND D
       PUSH H         ;THEN H
       MVI  C,11      ;GET CONSTAT WORD
       CALL CPM       ;CALL THE BDOS
```

BC/DE/HLレジスタをスタックに退避する。

Cレジスタに11を設定し、CP/MのシステムコールCONSTATを呼び出し、キーボードの接続状態を確認する。
```
       ORA  A         ;SET FLAGS
       JNZ  CI1       ;IF READY GET CHARACTER
       JMP  IDONE     ;RESTORE AND RETURN
CI1    MVI  C,1       ;GET CONIN WORD
       CALL CPM       ;CALL THE BDOS
```

システムコールの戻り値が0でなければ、CI1へジャンプし、文字を取得する。0であればIDONEへジャンプし、スタックへ退避しておいたBC/DE/HLレジスタを復元する。
```
       CPI  0FH       ;IS IT CONTROL-O?
       JNZ  CI2       ;NO, MORE CHECKING
       LDA  OCSW      ;CONTROL-O  FLIP OCSW
       CMA            ;ON TO OFF, OFF TO ON
       STA  OCSW      ;AND PUT IT BACK
       JMP  CHKIO     ;AND GET ANOTHER CHARACTER
```

入力された文字が0FHでなければ、CI2にジャンプする。0FHであればOSCWの内容を反転し、もう一文字取得するためCHKIOへジャンプする。
```
CI2    CPI  3         ;IS IT CONTROL-C?
       JNZ  IDONE     ;RETURN AND RESTORE IF NOT
       JMP  RSTART    ;YES, RESTART TBI
```
入力が03Hであれば、RSTARTへジャンプし、Tiny Basicをリスタートする。03Hでなければ、BC/DE/HLレジスタを復元し、呼び出し元へ戻る。
### OUTC/CRLF

OUTC(RST 2)ではAレジスタに格納されている文字を印字する。CRLFは復帰コード(0DH)と改行コード(0AH)を印字する。
```
CRLF:  EQU  0EH       ;EXECUTE TIME LOCATION OF THIS INSTRUCTION.
       MVI  A,0DH     ;*** CRLF ***
;* 
```

'CALL CRLF(CALL 000EH)'を実行した場合、Aレジスタに復帰コード(0DH)を入れて、そのままOUTCの処理へ進む。
```
       PUSH PSW       ;*** OUTC OR RST 2 *** 
       LDA  OCSW      ;PRINT CHARACTER ONLY
       ORA  A         ;IFF OCSW SWITCH IS ON
       JMP  OC2       ;REST OF THIS IS AT OC2
```

OUTCはRST 2で呼び出される。RST 2を実行すると、1842行目(アドレス0010H)から実行される。

Aレジスタ(印字する文字)をスタックに退避しておく。

出力スイッチ(OCSW)の内容を確認する。ORA命令でフラグを変化させておいて、OC2へジャンプする。
```
OC2    JNZ  OC3       ;IT IS ON
       POP  PSW       ;IT IS OFF 
       RET            ;RESTORE AF AND RETURN 
```

出力スイッチ(OCSW)が0であれば、文字を出力しない。Aレジスタを復旧し、呼び出し元に戻る。0でなければOC3へジャンプする。
```
OC3    POP  A         ;GET OLD A BACK
       PUSH B         ;SAVE B ON STACK
       PUSH D         ;AND D
       PUSH H         ;AND H TOO
       STA  OUTCAR    ;SAVE CHARACTER
       MOV  E,A       ;PUT CHAR. IN E FOR CPM
       MVI  C,2       ;GET CONOUT COMMAND
       CALL CPM       ;CALL CPM AND DO IT
```

Aレジスタ(文字データ)を復旧し、BCレジスタとDEレジスタ、HLレジスタをスタックに退避する。Aレジスタの内容はOUTCARに保管する。

CP/Mのシステムコールを使って印字を行う。Eレジスタに印字する文字、Cレジスタに文字出力(CONOUT)を表す2を設定し、CP/Mのシステムコールを呼び出す(CALL 0005H)。
```
       LDA  OUTCAR    ;GET CHAR. BACK
       CPI  0DH       ;WAS IT A 'CR'?
       JNZ  DONE      ;NO, DONE
       MVI  E,0AH     ;GET LINEFEED
       MVI  C,2       ;AND CONOUT AGAIN
       CALL CPM       ;CALL CPM
```

Aレジスタに印字した文字を復旧する。もし印字した文字が復帰コードであれば、改行コード(0AH)を印字する。
```
DONE   LDA  OUTCAR    ;GET CHARACTER BACK
IDONE  POP  H         ;GET H BACK
       POP  D         ;AND D
       POP  B         ;AND B TOO
       RET            ;DONE AT LAST
```

A/BC/DE/HLの各レジスタの内容を復旧し、呼び出し元に戻る。
### PRTSTG

テキストポインタ(DEレジスタ)で始まる文字列を印字する。終端文字(Aレジスタ)を見つけたら、印字を終了する。
```
PRTSTG MOV  B,A       ;*** PRTSTG ***
PS1    LDAX D         ;GET A CHARACTERr 
       INX  D         ;BUMP POINTER
       CMP  B         ;SAME AS OLD A?
       RZ             ;YES, RETURN 
       RST  2         ;ELSE PRINT IT 
       CPI  0DH       ;WAS IT A CR?
       JNZ  PS1       ;NO, NEXT
       RET            ;YES, RETURN 
```

文の末尾を示す文字(終端文字)をBレジスタに設定する。

テキストポインタ(DEレジスタ)が指す文字をAレジスタに読み込み、テキストポインタを一文字すすめる。

読み込んだ文字(Aレジスタの内容)が終端文字(Bレジスタの内容)と一致したら、印字を終了し、呼び出し元に戻る。違っていれば、OUTC(RST 2)を呼び出してAレジスタの内容を印字する。

印字した文字(Aレジスタの内容)と復帰コード(0DH)を比較し、一致すれば呼び出し元に戻る。一致しなければPS1に戻り、印字を継続する。
### PRTNUM

数字を印字する。
```
PRTNUM PUSH D         ;*** PRTNUM ***
       LXI  D,12Q     ;DECIMAL 
       PUSH D         ;SAVE AS A FLAG
       MOV  B,D       ;B=SIGN
       DCR  C         ;C=SPACES
       CALL CHKSGN    ;CHECK SIGN
       JP   PN1       ;NO SIGN 
       MVI  B,55Q     ;B=SIGN
       DCR  C         ;'-' TAKES SPACE 
```

テキストポインタ(DEレジスタ)をスタックに退避する。

DEレジスタに10(000AH)を設定し、スタックにプッシュする。

BレジスタにDレジスタの値(00H)を代入する。Bレジスタは符号を表すために使用する。

桁を表すCレジスタを一つ減らす。

CHKSGNを呼び出し、符号を確認する。正の値であればPN1へ進む。負の値であれば、Bレジスタに'-'を入れ、Cレジスタを一つ減らす。
```
PN1    PUSH B         ;SAVE SIGN & SPACE 
PN2    CALL DIVIDE    ;DEVIDE HL BY 10 
       MOV  A,B       ;RESULT 0? 
       ORA  C 
       JZ   PN3       ;YES, WE GOT ALL 
       XTHL           ;NO, SAVE REMAINDER
       DCR  L         ;AND COUNT SPACE 
       PUSH H         ;HL IS OLD BC
       MOV  H,B       ;MOVE RESULT TO BC 
       MOV  L,C 
       JMP  PN2       ;AND DIVIDE BY 10
```

PN1ではまず符号を表すBCレジスタをスタックに退避する。

DIVIDEを呼び出し、HLレジスタをDEレジスタ(10が入っている)で除算する。

商(BCレジスタ)が0であるかどうかを確認する。上位バイト(Bレジスタ)と下位バイト(Cレジスタ)のORを取って結果が0であれば、BCレジスタはすべてのビットが立っていない、つまり0であると判断できる。0であればすべての桁を処理したと判断し、PN3へジャンプする。

商が0でない場合、XTHL命令でHLレジスタ(剰余)とスタックの先頭にあるデータ(上位:符号、下位:桁数)を入れ替える。

Lレジスタ(桁数)を一つ減らして、HLレジスタ(上位:符号、下位:桁数)をスタックにプッシュする。

HLレジスタの内容をBCレジスタに代入し、PN2へ戻る。
```
PN3    POP  B         ;WE GOT ALL DIGITS IN
PN4    DCR  C         ;THE STACK 
       MOV  A,C       ;LOOK AT SPACE COUNT 
       ORA  A 
       JM   PN5       ;NO LEADING BLANKS 
       MVI  A,40Q     ;LEADING BLANKS
       RST  2 
       JMP  PN4       ;MORE? 
```

すべての桁を処理した場合、PN3へ移動する。

スタック先頭のデータ(上位:符号、下位:桁数)をBCレジスタに代入する。

桁数(Cレジスタ)を一つ減らす。

Cレジスタが負になっていなければ、OUTC(RST 2)を使って' '(20H)を印字し、PN4に戻る。

Cレジスタが負になっていれば、空白文字の印字が終わったと判断しPN5へジャンプする。
```
PN5    MOV  A,B       ;PRINT SIGN
       RST  2         ;MAYBE - OR NULL 
       MOV  E,L       ;LAST REMAINDER IN E 
```

Bレジスタから符号を取り出し、OUTC(RST 2)を使って印字する。

この時点ではHLレジスタに最後の除算による剰余、つまり数値の最上位桁が入っている。剰余は10より小さいので、下位バイトだけが有効な数字である。

この数値をEレジスタに代入する。これは、あとの処理でスタックからデータ(これまでの除算の剰余)をDEレジスタへ読み込むためである。
```
PN6    MOV  A,E       ;CHECK DIGIT IN E
       CPI  12Q       ;10 IS FLAG FOR NO MORE
       POP  D 
       RZ             ;IFF SO, RETURN 
       ADI  60Q		;ELSE CONVERT TO ASCII
       RST  2         ;AND PRINT THE DIGIT 
       JMP  PN6       ;GO BACK FOR MORE
```

下位バイト(Lレジスタ)をEレジスタに入れ、10と比較するが、最初は必ず10より小さな数であるので、一致しない。

スタックの先頭データ(次の桁の数値)をDEレジスタへ戻す。

RZ命令を実施するが最初は一致しないので次に進む。

剰余に'0'(30H)を加算し、文字へ変換し、印字する。

PN6へジャンプする。

2回目以降の比較で10と一致した場合、DEレジスタにはPRTNUMの最初でスタックにプッシュした数値(10)を読み出したことがわかる。スタックからテキストポインタをDEレジスタに読み出し、RZ命令を実行して呼び出し元に戻る。

### PRTLN

プログラムテキストを一行印字する。
```
PRTLN  LDAX D         ;*** PRTLN *** 
       MOV  L,A       ;LOW ORDER LINE #
       INX  D 
       LDAX D         ;HIGH ORDER
       MOV  H,A 
       INX  D 
       MVI  C,4Q      ;PRINT 4 DIGIT LINE #
       CALL PRTNUM
       MVI  A,40Q     ;FOLLOWED BY A BLANK 
       RST  2 
       SUB  A         ;AND THEN THE TEXT 
       CALL PRTSTG
       RET
```

テキストポインタ(DEレジスタ)の内容をHLレジスタにコピーする。PRTLNを呼び出すときのDEレジスタは行番号を指しているので、HLレジスタには行番号が設定される。

行番号は4桁である。Cレジスタに4を設定し、PRTNUMを呼び出す。

Aレジスタに' 'を設定し、OUTC(RST 2)を呼び出して' 'を印字する。

Aレジスタに0を設定し、文を印字する。
