---
title: Spacemacs on WSL
permalink: /docs/spacemacs-on-wsl/
toc: true
---
## はじめに
Windows 10の機能であるWSL(Windows Subsystem for Linux)上でSpacemacsを動かすための設定をまとめた。SpacemacsはXウインドウ上で動作させ、日本語入力にはWSL上で動くmozcを使用する。また、migemoによるローマ字検索の設定を行う。

先にWSL上でのEmacsを使ってみた感想を述べておくと、VirtualBox上のLinuxでEmacsを動かしたときより動作は重たく感じる。WindowsからEmacsを手軽に使用できるというメリットはあるものの、快適性は今ひとつだ。

## Ubuntuのインストール

Microsoft StoreからUbuntuをインストールし、Ubuntuを起動して次の作業を行う。

1. Emacs バージョン26が公開されている(非公式の)サイトをaptリポジトリに追加しておく。Ubuntuの公式パッケージにはバージョン25のEmacsが含まれているが、Spacemacsを使用するつもりなので、バージョンは26のほうが問題が少ない(何かの公開鍵が期限切れになっていることが原因のようだ。gnu-elpa-keyring-updateを一度インストールすれば良い)。
   ```
   sudo add-apt-repository ppa:kelleyk/emacs
   ```
2. ソフトウェアパッケージのデータベースを最新のものに更新する。
   ```
   sudo apt update
   ```
3. ソフトウェアを最新のものに更新する。
   ```
   sudo apt upgrade
   ```

## 日本語環境の設定

次に日本語環境の設定を行う。Xウインドウ用の日本語フォントもここで準備する。

1. 日本語のLanguage Packをインストールする。
   ```
   sudo apt install language-pack-ja
   ```

2. 日本語変換ソフトウェアであるmozcをインストールする。Windowsで動いているGoogle日本語変換を使う方法もあるようだが、WindowsにGoogle日本語変換を入れたくないので、WSL上でmozcを動かすことにする。emacs-mozcをインストールしようとすると、自動的にemacs25がインストールされてしまうため、emacs-mozc-binをインストールする。
   ```
   sudo apt install emacs-mozc-bin
   ```

3. Emacs内で半角ローマ字のまま日本語検索を行うためにmigemoをインストールしておく。

   ```
   sudo apt install cmigemo
   ```

画面表示用の日本語フォントにはRictyを使いたいので、Rictyの公式サイト(https://rictyfonts.github.io/)の手順に従ってフォントを生成する。

1. フォントの生成に必要なソフトウェア(fontforge)をインストールする。

   ```
   sudo apt install fontforge
   ```
2. Rictyの公式サイトから生成用のシェルスクリプト(ricty_generator.sh)を入手する。

3. Rictyを生成するために必要なフォントであるInconsolata(https://fonts.google.com/specimen/Inconsolata)とMigu 1M(http://mix-mplus-ipa.osdn.jp/)を入手する。

4. 作業用ディレクトリにricty_generator.shと入手したフォントのTTFファイルを置き、シェルスクリプトを実行する。

   ```
   sh ricty_generator.sh auto
   ```

5. フォント格納用のディレクトリを作成し、フォントをコピーする。

   ```
   mkdir -p ~/.local/share/fonts/Ricty
   mkdir ~/.local/share/fonts/Inconsolata
   mkdir ~/.local/share/fonts/migu-1m
   cp Ricty*.ttf ~/.local/share/fonts/Ricty
   cp Inconsolata*.ttf ~/.local/share/fonts/Inconsolata
   cp migu-1m*.ttf ~/.local/share/fonts/migu-1m
   ```

6. フォントキャッシュを更新する。

   ```
   fc-cache -fv
   ```

## Emacs(Spacemacs)のインストールと設定

1. Emacs(バージョン26)をインストールする
   ```
   sudo apt install emacs26
   ```

2. Spacemacsをインストールする。
   ```
   git clone https://github.com/syl20bnr/spacemacs ~/.emacs.d
   ```

3. Xウインドウ上でEmacsを動作させるために、bashの設定ファイル(~/.bashrc)に次の設定を追加する。
   ```
   export DISPLAY=localhost:0.0
   export LANG="ja_JP.UTF-8"
   export LIBGL_ALWAYS_INDIRECT=1
   ```

4. XサーバーソフトウェアVcxSrv(https://sourceforge.net/projects/vcxsrv/)を準備し、起動しておく。

5. bashの設定を確認するため、一度Ubuntuを起動し直してから、Emacsを立ち上げる。

6. Emacsを起動すると、Spacemacsのキー入力方式を尋ねられる。Emacsのほうが慣れているため、Holyモードを選択する。入力補完パッケージは、ivyのほうが少し軽いということだが、migemoの設定で問題が生じないhelmにする。

パッケージの自動インストールが終了した後、設定ファイル(~/.spacemacs)の編集を行う。

1. 表示用のフォントを設定する。変数dotemacs-default-fontを探し、次のように変更する。

   ```~/.spacemacs:dotspacemacs-default-font
   dotspacemacs-default-font '("Ricty"
                                 :size 16
                                 :weight normal
                                 :width normal
                                 :powerline-scale 1.4)
   ```

2. mozcとmigemoが読み込まれるように、変数dotspacemacs-additional-packagesにパッケージ名を記入する。

   ```~/.spacemacs:dotspacemacs-additional-packages
   dotspacemacs-additional-packages '(
                                        migemo
                                        mozc
                                        )
   ```

3. 次に関数dotspacemacs/user-configを次のように修正する。

   ```~/.spacemacs:dotspacemacs/user-config
   (defun dotspacemacs/user-config ()
     "Configuration function for user code.
   This function is called at the very end of Spacemacs initialization after
   layers configuration.
   This is the place where most of your configurations should be done. Unless it is
   explicitly specified that a variable should be set before a package is loaded,
   you should place your code here."
     (let*
         ((user-emacs-directory "~/.emacs.d/private/")
           (conf-list (list
                      "migemo-init.el"
                      )))
       (progn
         (add-to-list 'load-path (concat user-emacs-directory "lisp"))
         (dolist (conf conf-list)
           (load (concat user-emacs-directory "conf/" conf)))
         (and (equal window-system 'mac)
              (dolist (conf (list
                             "macosx-init.el"
                             ))
                (load (concat user-emacs-directory "conf/" conf))))
         (and (equal window-system 'w32)
              (dolist (conf (list
                             "w32-init.el"
                             ))
                (load (concat user-emacs-directory "conf/" conf))))
         (and (equal window-system 'x)
              (dolist (conf (list
                             "mozc-init.el"
                             "x11-init.el"
                             ))
                (load (concat user-emacs-directory "conf/" conf))))
         (and (null window-system)
              (dolist (conf (list
                             "term-init.el"
                             ))
                (load (concat user-emacs-directory "conf/" conf))))
         ))
     )
   ```

   関数dotspacemacs/user-configでは、~/.emacs.d/private/conf/の下にある設定ファイル(*.el)を実行する。最初に機種に依存しない設定を行い(10〜12行、15〜16行)、その後(17〜37行)、機種別の設定を行う。

4. migemoの設定ファイル(~/.emacs.d/private/conf/migemo-init.el)は次のようにしている。おそらく普通の設定だろう。

   ```~/.emacs.d/private/conf/migemo-init.el
   ;; -*-coding: utf-8-*-
   (when (and (executable-find "cmigemo")
              (require 'migemo nil t))
     (setq migemo-command "cmigemo")
     (setq migemo-options '("-q" "--emacs"))
     (setq migemo-dictionary "/usr/share/cmigemo/utf-8/migemo-dict")
     (setq migemo-user-dictionary nil)
     (setq migemo-regex-dictionary nil)
     (setq migemo-coding-system 'utf-8-unix)
     (load-library "migemo")
     (migemo-init)
     )
   ```

5. mozcの設定ファイル(~/.emacs.d/private/conf/mozc-init.el)のほうは下記のようにしている。変換キーを押せばmozcでのかな漢字変換モード、無変換キーを押せば半角入力モードになる。自分で作った記憶は無いのでどこかのWebサイトから拾ってきたコードだろう。

   ```~/.emacs.d/private/conf/mozc-init.el
   ;; Mozcの設定
   (when (and (executable-find "mozc_emacs_helper")
              (require 'mozc nil t))
     (set-input-method 'japanese-mozc)
     (inactivate-input-method)
     (setq mozc-candidate-style 'echo-area)
     (global-set-key [henkan]
                     (lambda ()
                       (interactive)
                       (activate-input-method 'japanese-mozc)))
     (global-set-key [muhenkan]
                     (lambda ()
                       (interactive)
                       (inactivate-input-method)))
     (global-set-key [zenkaku-hankaku]
                     (lambda ()
                       (interactive)
                       (toggle-input-method)))
     (defadvice mozc-handle-event (around intercept-keys (event))
       "Intercept keys muhenkan and zenkaku-hankaku,
        before passing keys to mozc-server (which the function mozc-handle-event does),
        to properly disable mozc-mode."
       (if (member event (list 'zenkaku-hankaku 'muhenkan))
           (progn (mozc-clean-up-session)
                  (toggle-input-method))
         (progn ad-do-it)))
     (ad-activate 'mozc-handle-event)
     )
   ```

## Windowsの設定(オプション)

Emacsしか使わないなら、Windowsのデスクトップ上でアイコンをダブルクリックすれば、Emacsが立ち上がるようにしておいてもいいだろう。その方法も残しておく。

1. VcXsrvがログイン時に立ち上がるように、スタートアップにVcXsrvの設定ファイルを入れておく。設定ファイルの拡張子はxlaunchにしておく必要がある。

2. 次のVBScriptを作成する(Emacs.vbs)。
   ```
   set objWshShell = WScript.CreateObject("WScript.Shell")
   objWshShell.Run "C:\Windows\System32\wsl.exe -u %USERNAME% env DISPLAY=localhost:0.0 LANG==""ja_JP.UTF-8"" LIBGL_ALWAYS_INDIRECT=1 emacs", vbHide
   ```
上のVBScriptをダブルクリックすれば、Emacsが立ち上がる。アイコンを変えたい場合は、このVBScriptのショートカットを作成し、ショートカットのアイコンを変更する(VBScriptのアイコンを直接変更することはできない)
