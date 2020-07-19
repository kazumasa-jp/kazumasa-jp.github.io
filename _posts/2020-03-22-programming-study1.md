---
title: mu4e
author: Kazumasa
layout: post
tags: [Lisp, mu4e]
---
メールの読み書きも使い慣れたエディタでできるのなら、それに越したことはないと思う。Spacemacsのレイヤーにはmu4eが準備されているので、mu4eを使ってみることにした。

Gmailからメールを転送し、mu4eでメッセージを見てみると、マルチパートのメールで文字コードにISO-2022-JPを指定してあるメールがUTF-8に変換されない。最初はmu4e-messages.elを直接編集していたが、mu4e-message-body-rewrite-functionsに変換用の関数を登録しておけば良いことがわかった。次のコードを起動時に読み込ませるようにしている。
```elisp
(defun my-mu4e-decode-body-text (msg txt)
  (let* ((text-coding
          (mm-charset-to-coding-system
           (assoc-default
            "charset"
            (mu4e-message-field msg :body-txt-params)))))
    (if mu4e~message-body-html txt
      (decode-coding-string txt text-coding))))
(add-to-list 'mu4e-message-body-rewrite-functions 'my-mu4e-decode-body-text t)
```
これでmu4e-view-prefer-htmlをnilにしておけば大抵の場合は大丈夫なのだが、HTMLメールの中には正しく表示できないものがある。html→text変換のプログラムをいくつか試してみたが何故かうまく行かない。mu4e側の問題なのだろうか。

そういえば、ちゃんとEmacs Lispの勉強をしたことがなかったので、"An Introduction to Programming in Emacs Lisp"を読み始めた。まだ"1 List Processing"を読み終わったところだが、知らなかったこともいくつかある。"if"はspecial formで"when"はマクロというのはいいとして、"defun"もマクロになっているとは。シンボルへ関数と値の両方を同時に設定できることも意識していなかった。
あとはLispについての説明もリストの説明から始めるのではなく、アトムの説明から始めたほうが楽なのではと思った。Lispの世界がどうなっているか、最小単位のアトムから始めたらどうだろう。そちらのほうが何も知らない人にとってはわかりやすくなったりしないだろうか。
