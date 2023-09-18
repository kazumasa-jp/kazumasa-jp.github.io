---
title: Ledger(ledger-cli)入門
permalink: /docs/ledger-beginner/
toc: true
---
## はじめに

Ledgerは複式簿記の考え方でお金の出入りを管理する会計システムです。グラフィカルなユーザーインターフェースは持っていません。英語のドキュメントは充実しているようですが、わかりにくいところがあります。自分の英語力や会計についての知識不足もありますが、自分なりの理解でLedgerの使い方をまとめたいと思います。

## 仕訳帳(journal)の作成

### 取引の記入

Ledgerが処理するファイルを仕訳帳(journal)と呼びます。仕訳帳には日々の取引(transaction)の内容を記述してあります。次は取引の記入例です。

```
9/29 (1023) Pacific Bell
    Expenses:Utilities:Phone                   $23.00
    Assets:Checking                           $-23.00
```

一行目には取引の日付(9/29)と取引コード(1023)、そして受取人名(Pacific Bell)が書かれています。これを取引行と呼ぶことにします。取引コードは必ずしも書く必要はありません。日付についても2018/9/29のように年を指定することもできます。

二行目以降は仕訳の内容が書かれています。これを仕訳行(posting)と呼ぶことにします。仕訳行には勘定項目(account item)と金額が書かれていますが、仕訳行の先頭は半角スペースを一つ以上空けておく必要があります。

最初の仕訳行の勘定項目(account item)は`Expenses:Utilities:Phone`で、対応する金額は$23.00です。正の値なので借方であることを意味します。

次の仕訳行の勘定項目は「Assets:Checking」で、対応する金額は$-23.00です。負の値になっているので、こちらは貸方であることを意味しています。

勘定項目は「大項目:中項目:小項目」のように階層化されています。階層の深さは3つ以上でも構いません。仕訳行は何行になっても構いませんが、借方と貸方を合わせた資産の合計は0に釣り合わなければなりません。しかし、一行だけであれば金額を省略することが可能です。その場合、自動的に取引の総額が0になるような値が設定されます。

```
9/29 (1023) Pacific Bell
    Expenses:Utilities:Phone                   $23.00
    Assets:Checking
```

受取人名や勘定項目には日本語を使うこともできるようです(「支出:光熱費:電気」など)。ただし、文字コードに制約があるかもしれません。

```
9/29 (1023) パシフィックベル
     支出:光熱費:電話                      $23.00
     資産:当座預金                         $-23.00
```

### 開始残高の設定

Ledgerを使い始める段階で、すでにいくらかの資産を持っているはずです。その資産を開始残高としてLedgerへ取引の形で記録します。例えば、当座預金口座(Assets:Checking)に$100.00を持っていたとします。

```
10/2  Opening Balance
    Assets:Checking                         $100.00
    Equity:Opening Balances
```

複式簿記の考え方に従えば、取引は釣り合いが取れていなければなりません。釣り合いを取るための勘定項目がエクイティ口座(Equity:Opening Balances)です。

エクイティ口座は残高レポートでは大きな負の値として表示されますが、その絶対値がLedger開始時のあなたの資産です。この絶対値を総資産が超えている場合、Ledgerを使い始めてから資産が増加したことを意味します。

## レポートの作成

### コマンドラインでの文法

Ledgerはいくつかのレポート作成コマンドとそれらのコマンドからの出力を変形するための多くのオプションを持っています。Ledgerの基本文法は、

```
$ ledger [OPTIONS...] COMMAND [ARGS...]
```

命令語(COMMAND)のあと、引数(ARGS...)が続きます。大抵の命令語では、引数は正規表現です。Ledgerは、正規表現に適合した項目に関係した出力を行います。

正規表現は通常、勘定項目に適合します。取引の受取人に関連した出力を行うには、正規表現の前にpayeeという語を挿入するか、正規表現の先頭に@をつけます。

例えば、次のbalance命令は家賃(rent)、食費(food)、映画(movie)についての勘定項目の残高を示しますが、受取人がfreddieになっているものだけを抽出します。

```
$ ledger -f sample.dat balance rent food movies payee freddie
$ ledger -f sample.dat balance rent food movies @freddie
```

上の例ではledgerで処理する仕訳帳ファイルが指定してありませんが、仕訳帳ファイルはオプションの--fileまたは-fで指定するか、環境変数LEDGER_FILEに予め設定しておく必要があります。--fileオプションに渡すファイル名には、標準入力(/dev/stdin)の短縮形として「-」を使うことができます。

### レポートの種類

Ledgerが作成するレポートには、残高レポート、取引レポート、決済レポートがあります。

ここでは次の簡単な仕訳帳ファイル(sample.dat)を考えます。

```
2011/03/01 * Opening Balance
    Assets:Wallet                         $1000.00
    Assets:Checking                       $2000.00
    Equity:Opening Balances
2011/03/15 * Trader Joe's
    Expenses:Groceries                    $100.00
    Assets:Wallet
2011/03/17   Whole Food Market
    Expenses:Groceries                    $75.00
    Liabilities:Credit Card
2011/03/19 *  Pacific Bell
    Expenses:Utilities:Phone              $23.00
    Assets:Checking
```

- 2011/03/01の時点で、開始残高として財布(Assets:Wallet)に$1000.00、当座預金に$2000.00を持っています。
- 2011/03/15にTrader Joe'sという店で$100.00の食料品(Expenses:Groceries)を現金(Assets:Wallet)で購入しています。
- 2011/03/17にWhole Food Marketで$75.00分の食料をクレジットカード(Liabilities:Credit Card)で買っています。
- 2011/03/19にPacific Bellによって当座預金から電話料金が引き落とされています。

これを使ってLedgerがどのようなレポートを出力するか見ていきましょう。

#### 残高レポート(balance report)

すべての勘定項目の残高を表示するためには、次のコマンドを実行します。

```
$ ledger -f sample.dat balance
```

出力は次のようになります。
```
            $2877.00  Assets
            $1977.00    Checking
             $900.00    Wallet
           $-3000.00  Equity:Opening Balances
             $198.00  Expenses
             $175.00    Groceries
              $23.00    Utilities:Phone
             $-75.00  Liabilities:Credit Card
--------------------
                   0
```

最初の行は「当座預金(Assets:Checking)」と「財布(Assets:Wallet)」が属する「資産(Assets)」グループの残高を表しています。

4行目のエクイティ口座(Equity:Opening Balances)は先に述べたように負の値を持ち、その絶対値は開始残高を表しています。「資産(Assets)」を使って買い物をしたので、「資産(Assets)」グループの残高は開始残高を下回っています。

5行目はこの仕訳帳ファイル(sample.dat)内の「支出(Expenses)」を表しています。6行目の「食料品(Groceries)」は、Trader Joe'sとWhole Food Marketで購入した食料品の総額です。7行目はPacific Bellに支払った「電話料金(Utilities:Phone)」です。

8行目はクレジットカードによる支払い額で、「負債(Liabilities)」として扱われるため、負の値になっています。

最終行は、金額の総計が0になっており、すべての勘定項目の釣り合いが取れていることを表しています。

コマンドの後ろに勘定項目名を指定すると、表示内容を絞り込むことができます。例えば、資産(Assets)と負債(Liabilities)のみを表示させてみましょう。

```
$ ledger -f sample.dat balance Assets Liabilities
```

実行結果は下記になります。
```
            $2877.00  Assets
            $1977.00    Checking
             $900.00    Wallet
             $-75.00  Liabilities:Credit Card
--------------------
            $2802.00
```

最終行は、「資産(Assets)」と「負債(Liabilities)」の和、つまり純資産を表しています。

#### 取引レポート

すべての取引と資産の累計を表示させるには、次のコマンドを実行します。

```
$ ledger -f sample.dat register
```

Ledgerは次を表示します。
```
11-Mar-01 Opening Balance       Assets:Wallet              $1000.00     $1000.00
                                Assets:Checking            $2000.00     $3000.00
                                Equit:Opening Balances    $-3000.00            0
11-Mar-15 Trader Joe's          Expenses:Groceries          $100.00      $100.00
                                Assets:Wallet              $-100.00            0
11-Mar-17 Whole Food Market     Expenses:Groceries           $75.00       $75.00
                                Liabilitie:Credit Card      $-75.00            0
11-Mar-19 Pacific Bell          Expens:Utilities:Phone       $23.00       $23.00
                                Assets:Checking             $-23.00            0
```

各行には2つの金額が表示されています。最初の金額は正の金額なら貸方、負の金額なら借方を表し、二番目の金額は資産の累計です。

コマンドラインに勘定項目を追加することで、確認したい勘定項目だけを表示させることができます。

```
$ ledger -f drewr3.dat register Groceries
---
11-Mar-15 Trader Joe's          Expenses:Groceries          $100.00      $100.00
11-Mar-17 Whole Food Market     Expenses:Groceries           $75.00      $175.00
```

特定の受取人にだけ関係する取引を表示したい場合、「payee」か「@」を使用します。

```
$ ledger -f drewr3.dat register payee "Organic"
---
11-Mar-15 Trader Joe's          Expenses:Groceries          $100.00      $100.00
                                Assets:Wallet              $-100.00            0
```

#### 決済レポート

例えば、クレジットカード払いなどは、クレジットカードを使用した日から実際に決済が完了するまでに何日かかかることがありますが、クレジットカードを使用した時点でお金は使われたと考えておくべきです。

しかし、まだ決済がすべて完了していると仮定したときの残高と、実際に決済が完了している残高との比較は、債務を把握する上で有益です。

決済が完了した取引には、取引の日付と受取人の間にアスタリスク(「*」)を書きます。
決済が済んでいない残高を表示するためには、次のコマンドを入力します。

```
$ ledger -f sample.dat cleared
---
        $2877.00            $2877.00                 Assets
        $1977.00            $1977.00    11-Mar-19      Checking
         $900.00             $900.00    11-Mar-15      Wallet
       $-3000.00           $-3000.00    11-Mar-01    Equity:Opening Balances
         $198.00             $123.00                 Expenses
         $175.00             $100.00    11-Mar-15      Groceries
          $23.00              $23.00    11-Mar-19      Utilities:Phone
         $-75.00                   0                 Liabilities:Credit Card
----------------    ----------------    ---------
               0                   0             
```

最初の列には決済がすんでいない取引も含まれた残高が表示されていますが、次の列はすでに決済が完了している取引のみから算出した残高が表示されています。3番目の列は決済が完了している取引が最後に行われた日付です。

### 期間の指定

レポートの集計期間を限定するには、次のオプションを使います。

+ --begin DATE
+ -b DATE 

  DATEで表される日付以降の取引を使ってレポートを作成します。残高レポートはこの日以前の残高を0として累計を計算します。

+ --end DATE

+ -e DATE 

  DATEで表される日付より後の取引をレポート作成には使用しません。

+ --current

  現在の日付以前の取引に集計範囲を限定します。現在の日付は--nowオプションを使うことで、実際とは違う日に設定することも可能です。

+ --now DATE 

  現在の日付を定義します。日付を表す引数DATEは、何も指定しなければ、"4桁の西暦/月/日"です(2018/1/31など)。DATEの書式を変えるには、次のオプションを使います。

+ --input-date-format DATE_FORMAT

  日付を表すパラメータ(DATE)の書式をDATE_FORMATに設定します。例えば、日付の書式を"日/月/2桁の西暦"とするには、--input-date-format '%d/%m/%y'とします。

### 取引の小計

#### 期間単位での小計

月単位、週単位など一定期間で取引をグループ化し、その小計を取るには、次のオプションを使います。

+ --daily

+ -D 

  日単位で小計を取ります。

+ --weekly

+ -W 

  週単位で小計を取ります。

+ --monthly

+ -M 

  月単位で小計を取ります。

+ --quarterly 

  四半期単位で小計を取ります。

+ --yearly

+ -Y 

  年単位で小計を取ります。

+ --dow 

  曜日単位で小計を取ります。

#### その他の単位での小計

+ --by-payee

+ -P 

  受取人毎に小計を取ります。

+ --subtotal

+ -s 

  台帳レポートでも残高レポートと同様に勘定項目毎に小計を取ります。

### 正規表現の使用

Ledgerが文字列を必要とする箇所では、一般的な正規表現を使うことができます。
次の例はCrで始まるすべての勘定項目を探します。Crで始まる勘定項目はありません。

```
$ ledger balance -f sample.dat ^Cr
```

Crを含む勘定項目を探すには次のようにします。

```
$ ledger balance -f sample.dat Cr
---
             $-75.00  Liabilities:Credit Card
```

もし特定の勘定項目から特定の受取人に支払った額を知りたければ、次のようにすれば、財布(Assets:Wallet)からTrader Joe'sへの支払額を調べることができます。

```
$ ledger balance --limit 'account=~/Wallet/' --limit 'payee=~/Joe/'
```

### レポート作成コマンドの例

次のコマンドは昨年10月(last oct)以降のすべての支出(Expenses)を表示します。

```
$ ledger -b "last oct" -S T balance ^Expenses
```

オプション-Sは出力のソートを指示します。TはTOTALを意味しており、総計で並べ替えることを意味しています。

次のコマンドは、月毎に支出をまとめます。

```
$ ledger -M --period-sort "(amount)" register ^Expenses
```

オプションの--period-sort "(amount)"は各月の支出を金額で並べ替えます。

逆にどのような手段で支払っているかを知りたいかもしれません。その時は次のコマンドを使用します。

```
$ ledger -M --period-sort "(amount)" --related register ^Expenses
```

オプションの--relatedは支出と対になる、つまり支払いに使った勘定項目を表示します。

もし、MasterCardでいくら支払ったかを知りたいだけであれば、--displayオプションを使う必要があります。registerコマンドの後には^Expenses(支出)を置いて計算範囲を決定する必要がある一方で、--relatedオプションで表示される支払いに使った勘定項目からMasterCardを抽出したいからです。具体的には次のようになります。

```
$ ledger -M --related --display 'account=~/MasterCard/' register ^Expenses
```

もし、下の階層の勘定項目をまとめてしまいたい場合は、--collapseオプションを使います。

```
$ ledger -M --collapse register ^Expenses
```

## 環境設定

Ledgerは多くのコマンドラインオプションを持っていますが、環境変数や初期設定ファイルに予め設定しておくことができます。

次は環境変数の例です。

- LEDGER_FILE

  Ledgerの仕訳帳ファイルを指定します。

- LEDGER_INIT_FILE

  Ledgerの初期設定ファイルを指定します。何も指定していなければ、「~/.ledgerrc」が使われます。

- LEDGER_PRICE_DB

  価格履歴ファイルを指定します。

初期設定ファイルは、環境変数LEDGER_INIT_FILEや--init-fileオプションで指定しなければ、~/.ledgerrcが使用されます。初期設定ファイルにはコマンドラインオプションを一行に一つ書きます。

```
--f ledger.dat
--input-date-format "%m/%d/%Y"
--price-db ~/finance/.pricedb
--wide
```

初期設定ファイルの設定内容よりも、環境変数やコマンドラインオプションで指定した内容のほうが優先されます。
