---
title: Dell Inspiron 11 3000へのArch Linuxのインストール(2020年3月版)
permalink: /docs/installing-archlinux/
toc: true
---
## 準備

### Arch Linuxインストール用USBメモリの作成

インストール用USBメモリはMacで作成した。

1) Arch Linuxのミラーサイトから次のファイルをダウンロードする。
   - archlinux-yyyy.mm.dd-x86_64.iso
   - archlinux-yyyy.mm.dd-x86_64.iso.sig
   - md5sums.txt
   - sha1sum.txt
   ここで、yyyyは年、mmは月、ddは日である。今回は2020年03月01日のイメージを使用した。
2) ファイルの正当性を確認する。
   ```
   $ gpg --keyserver-options auto-key-retrieve --verify archlinux-yyyy.mm.dd-x86_64.iso.sig
   $ md5 -r archlinux-yyyy.mm.dd-x86_64.iso | diff -b - md5sums.txt
   $ shasum -a 1 archlinux-yyyy.mm.dd-x86_64.iso | diff -b - sha1sum.txt
   ```
3) ファイルの正当性が確認できたら、イメージをUSBメモリに書き込む。
   ```
   $ diskutil list
   $ diskutil unmountDisk /dev/diskX
   $ dd if=./archlinux-yyyy.mm.dd-x86_64.iso of=/dev/rdiskX bs=1m
   ```

### 回復ドライブの作成

ここからはInspiron 11 3000での作業になる。

Linuxのインストール作業に入る前にプリインストールされているWindowsの回復ドライブを作成する。
Inspiron 11では、下記の5の段階で、8GB以上のUSBメモリが必要というメッセージを出していたが、パッケージに8GBと表示されているものでは若干容量が足りないので16GBのものが必要だ。

1) スタートメニューから、「Windows システムツール」を選ぶ。
2) 表示されたサブメニューから「コントロールパネル」を選び、コントロールパネルを起動する。
3) 「表示方法」が「大きいアイコン」または「小さいアイコン」になっているときは「回復」を選択し、「カテゴリ」になっている場合は、「システムとセキュリティ」→「セキュリティとメンテナンス」→「回復」を選ぶ。
4) 「回復ドライブの作成」を選ぶ。
5) 「このアプリがデバイスに変更を加えることを許可しますか？」(このアプリとは回復メディア作成ツールのこと)と表示されるので、「はい」を選ぶ。
6) 「システムファイルを回復ドライブにバックアップします」にチェックを入れて次へ進む
7) しばらく待つと回復ドライブに必要な容量が表示される。
8) 必要な容量を持っているUSBメモリを接続すると「使用可能なドライブ」として表示される。
9) 「次へ」を押すとドライブ上のすべてのデータが消去されると警告される。問題なければ「作成」を押す。
10) 回復ドライブの作成にはかなりの時間がかかる。「回復ドライブの準備ができました」と表示されたら、「完了」をおして回復ドライブの作成を終了する。

### Linux用領域の確保

WindowsとのDual Boot環境を作成する場合は、Windows用のパーティションを縮小してLinux用の領域を確保する必要がある。ここではArch Linuxは32GBのシングルパーティションにインストールすることにする。

1) [Windows]キーと[X]キーを同時に押す。
2) 表示されたメニューから「ディスクの管理」を選ぶ。
3) 「ディスクの管理」が表示されたら、Cドライブを右クリックし、「ボリュームの縮小」を選ぶ。
4) しばらく待つと「C:の縮小」が表示されるので、「縮小する領域のサイズ(MB)」に「32000」と入力する。

### 高速スタートアップの無効化

Windowsの高速スタートアップ機能は、シャットダウン時にユーザーセッションのみを終了させ、カーネルセッションは休止状態にすることで、次回起動時にドライバの初期化にかかる時間を節約する。基本的には有用な機能だが、次の起動までにハードウェアの変更があった場合は不具合を生じる可能性もある。Linuxとのデュアルブート環境ではEFIシステムパーティションを共有するので、高速スタートアップを無効にすることが推奨されている。

1) [[回復ドライブの作成]]と同様の手順で「コントロールパネル」を起動する。
2) 「システムとセキュリティ」→「電源オプション」を選ぶ。
3) 「電源オプション」が表示されたら、「電源ボタンの動作の選択」を選ぶ。
4) 「システム設定」が表示されるので、「現在利用可能ではない設定を変更します」を選ぶ。
5) 「シャットダウン設定」欄にある「高速スタートアップを有効にする」のチェックボックスをオフにする。

### ファームウェアの設定(Secure Boot/Fast Boot/Legacy Option Rom)

Inspiron 11にはUEFIが搭載されている。UEFIは従来のBIOSに代わるPC用のファームウェアで、

- 2テラバイトを超える大きさのディスクからのブート
- Secure Boot
- Fast Boot

などの機能を備えている。

Secure Bootはブートコードが改ざんされていないことを保証する機能で、Arch LinuxはSecure Bootに対応しているが、インストール時は無効にしておく。

Fast Bootは起動時のレガシーデバイスのスキャンを省略することで起動を高速化する機能で、Windowsの高速スタートアップとは別の機能の機能である。有効にしていても問題はなさそうだが、念の為にこれも無効にしておく。

また、Inspiron 11のUEFI設定メニューには「Load Legacy Option Rom」という項目があるが、ここが「Enabled」になっているとArch LinuxがSSDから起動しないので、ここもオフにする。

1) 起動時にDellのロゴが出ているとき、[F2]キーを押し、UEFIの設定画面に入ることができる。
2) [←]、[→]キーでタブを切り替え、「Boot」設定画面を表示させる。
3) [↑]、[↓]キーで項目を選択し、スペースで設定を変更する。「Fast Boot」、「Secure Boot」、「Load Legacy Option Rom」をそれぞれ「Disabled」に変更する。
4) 「Exit」設定画面で設定を保存し、再起動する。

## 基本インストール

ここでは、Arch LinuxをPCにインストールし、ワイヤレス・ネットワークへの接続設定、ユーザーの追加までを行う。

### インストール用USBメモリによる起動

USBメモリからブートするには、ブートデバイス選択画面を表示させる必要がある。

1) USBメモリを接続した状態で再起動する。
2) Dellのロゴが出ているときにF12キーを押す。
3) USBメモリを選択し、Arch Linuxを起動する。

### キーボードの設定

インストール作業に先立ち、日本語キーボード用のキーマップをロードする。

```
root@archiso ~ # loadkeys jp106
```

### 無線LANの設定

Inspiron 11には有線LANインターフェースがついていないので、ベースパッケージのインストール前に無線LANの設定をする必要がある。SSIDをステルスモードにしている場合について記載する。

1) 無線LANのインターフェース名を調べる。wlp1s0という名前だということがわかる。

   ```
   root@archiso ~ # ip link
   1: lo: <LOOPBACK, ...
   2: wlan0: <BROADCAST, ...
   ```

2) 無線LANインターフェースを有効化する。

   ```
   root@archiso ~ # ip link set wlan0 up
   ```

3) wpa_supplicantの設定ファイル(/etc/wpa_supplicant.conf)を作成する。

   ```
   root@archiso ~ # echo 'ctrl_interface=/run/wpa_supplicant' > /etc/wpa_supplicant.conf
   root@archiso ~ # wpa_passphrase MYSSID passphrase >> /etc/wpa_supplicant.conf
   ```

4) 設定ファイル(/etc/wpa_supplicant.conf)にステルスアクセスポイントを検索するためのオプション(scan_ssid=1)を追加する。emacsはインストールされていないため、編集にはviやnanoを使用する。

   ```
   root@archiso ~ # vi /etc/wpa_supplicant.conf
   ---
   ctrl_interface=/run/wpa_supplicant
   network={
       ssid="MYSSID"
       #psk="passphrase"
       psk=59e0d07fa4c7741797a4e394f38a5c321e3bed51d54ad5fcbd3f84bc7415d73d
       scan_ssid=1
   }
   ```

5) ネットワークに接続する。

   ```
   root@archiso ~ # wpa_supplicant -B -D nl80211 -c /etc/wpa_supplicant.conf -i wlan0
   root@archiso ~ # dhcpcd -A wlan0
   ```

6) ネットワーク接続を確認する

   ```
   root@archiso ~ # ping www.archlinux.org
   ```

### 時刻の設定

ネットワークに繋がったので、NTPを使ってシステムクロックを合わせる。

```
root@archiso ~ # timedatectl set-ntp true
```

### インストール先パーティションの作成

ここからは、Dual Boot環境にしない構成で説明する。Dual Boot環境の場合は必要に応じてディスク名(sdaX)を読み替えれば良い。

Linux用のパーティションをgdiskを使って作成する。作成結果は下記の通りで、Linux用のパーティションは/dev/sda3になる。

```
root@archiso ~ # gdisk /dev/sda
root@archiso ~ # gdisk -l /dev/sda
---
GPT fdisk (gdisk) version 1.0.5
...
Number  Start (sector)    End (sector)  Size      Code  Name
   1            2048         1026047   500.0 MiB  EF00  EFI System partition
   2         1026048         1288191   128.0 MiB  0C01  Microsoft reserved ...
   3         1288192       233224191   110.6 GiB  8300  Linux filesystem
   4       233224192       250068991   8.0 GiB    2700
---
root@archiso ~ # mkfs.ext4 /dev/sda3
```

### パーティションのマウント

インストール先のパーティションを/mntへマウントする。

```
root@archiso ~ # mount /dev/sda3 /mnt
root@archiso ~ # mount /dev/sda1 /mnt/boot
```

### ベースパッケージのインストール

Linux用のパーティション(/mnt)にベースパッケージをインストールする。

1) /etc/pacman.d/mirrorlistを修正し、日本のサイトを上位に持ってくる。
   ```
   root@archiso ~ # vi /etc/pacman.d/mirrorlist
   ---
   ##
   ## Arch Linux repository mirrorlist
   ## Filtered by mirror score from mirror status page
   ## Generated on 2020-03-01
   ##

   ## Japan
   Server = ...
   ## Japan
   Server = ...
   ## Japan
   Server = ...
   ....
   ```

2) ベースパッケージをインストールする

   ```
   root@archiso ~ # pacstrap /mnt base linux linux-firmware
   ```

### 初期の起動設定

SSDからのブートを可能にするために/mntにマウントしたパーティションに起動時の設定を書き込む。

1) fstabを作成する

   ```
   root@archiso ~ # genfstab -U /mnt >> /mnt/etc/fstab
   ```

2) ルートディレクトリを変更する

   ```
   root@archiso ~ # arch-chroot /mnt
   ```

3) Time Zoneの設定を行う

   ```
   [root@archiso /]# ln -s /usr/share/zoneinfo/Asia/Tokyo /etc/localtime
   [root@archiso /]# hwclock --systohc --utc
   ```

4) arch-chroot後の環境にはエディタやネットワーク関連のツールが入っていないため、ここでインストールしておく

   ```
   [root@archiso /]# pacman -S vi netctl wpa_supplicant dhcpcd
   ```

4) Localeの設定を行う。

   1. /etc/locale.genを修正し、en_US.UTF-8 UTF-8とja_JP.UTF-8 UTF-8のコメントを外す。

      ```
      [root@archiso /]# vi /etc/locale.gen
      ---
      # Configuration file for locale-gen
      #
      ...
      en_US.UTF-8 UTF-8
      ...
      ja_JP.UTF-8 UTF-8
      ...
      ```

   2. Localeを生成する

      ```
      [root@archiso /]# locale-gen
      ```

   3. デフォルトのLocaleを設定する。コンソール画面では英語しか表示されないため、デフォルトのLocaleは英語にする

      ```
      [root@archiso /]# echo LANG=en_US.UTF-8 > /etc/locale.conf
      ```

   4. 起動時にキーボード設定が反映されるようにしておく。

      ```
      [root@archiso /]# echo KEYMAP=jp106 > /etc/vconsole.conf
      ```

5) ネットワーク設定を行う

   1. Host名の設定を行う。以下、「myhostname」は自分のPCの名前に置き換えること。

      ```
      [root@archiso /]# echo myhostname > /etc/hostname
      ```

   2. /etc/hostsの修正を行う。

      ```
      [root@archiso /]# vi /etc/hosts
      ---
      127.0.0.1	localhost
      ::1		localhost
      127.0.1.1	myhostname.localdomain	myhostname
      ```
6) Ram FSを初期化する

   ```
   [root@archiso /]# mkinitcpio -p linux
   ```

7) パスワードを変更する

   ```
   [root@archiso /]# passwd
   ```

### ブートローダーのインストール

ブートローダーの選択肢はいくつかあるようだが、簡単に設定できそうなsystemd-bootを使うことにする。

1) EFI変数にアクセスできることを確認する

   ```
   [root@archiso /]# pacman -S efivar
   [root@archiso /]# efivar -l
   ```

2) UEFIパーティション(/dev/sda1)を/bootにマウントする。

   ```
   [root@archiso /]# mount /dev/sda1 /boot
   ```

3) systemd-bootブートローダーをインストールする

   ```
   [root@archiso /]# bootctl --path=/boot install
   ```

4) 一旦、UEFIパーティション(/dev/sda1)を別のディレクトリにマウントする

   ```
   [root@archiso /]# umount /boot
   [root@archiso /]# mount /dev/sda1 /mnt
   ```

5) initramfs-linux.img、vmlinuz-linuxをUEFIパーティションにコピーする。

   ```
   [root@archiso /]# cp /boot/initramfs-linux.img /mnt
   [root@archiso /]# cp /boot/vmlinuz-linux /mnt
   ```

6) Arch Linuxのエントリーを作成する。options行のrootオプションにはルートパーティションのPARTUUIDを指定する。PARTUUIDはblkidコマンドで調べることができる。

   ```
   [root@archiso /]# blkid /dev/sda3
   [root@archiso /]# vi /mnt/loader/entries/arch.conf
   ---
   title Arch Linux
   linux /vmlinuz-linux
   initrd /initramfs-linux.img
   options root=PARTUUID=xxxxxxxxxxxxxxxxxxxx rw
   ```

7) 作成したエントリーをブートローダーに登録する。設定ファイル中の"default arch"のarchは"arch.conf"を指している。"auto-entries no"を入れておくことで、Windows用のBootメニューが表示されなくなる。

   ```
   [root@archiso /]# vi /mnt/loader/loader.conf
   ---
   default arch
   timeout 3
   auto-entries no
   ```

8) EFI変数をクリアする。これを実施していない場合、loader.conf中のデフォルト値指定(default arch)が反映されないことがある。

   ```
   [root@archiso /]# bootctl set-default ""
   ```

9) PCを再起動し、SSDから立ち上がることを確認する。

   ```
   [root@archiso /]# umount /mnt
   [root@archiso /]# exit
   root@archiso ~ # reboot
   ```

### 再起動とネットワークサービスの登録

起動時にネットワークサービスが起動されるように設定する。rootでログインし、次の作業を行う。

1) [[無線LANの設定]]と同じ手順で、wpa_supplicantの設定ファイルを作成する。/etc/wpa_supplicant/というディレクトリがあるので、wpa_supplicant.confはこのディレクトリ内におくことにする。

2) 他のユーザーから読み書きできないように設定ファイルのモードを変更する。

   ```
   [root@myhostname ~]# chmod 600 /etc/wpa_supplicant/wpa_supplicant.conf
   ```

3) 無線LANインターフェースの名前を確認しておく。インストールUSBで起動したときとは名前が違っていることがあるため、注意すること

   ```
   [root@myhostname ~]# ip link
   1: lo: <LOOPBACK, ...
   2: wlp1s0: <BROADCAST, ...
   ```

4) netctlのプロファイルを作成する。/etc/netctl/examples/wireless-wpa-configをもとに修正を行う。

   ```
   [root@myhostname ~]# cp /etc/netctl/examples/wireless-wpa-config /etc/netctl/wireless-wpa-config
   [root@myhostname ~]# vi /etc/netctl/wireless-wpa-config
   ---
   Description='A wpa_supplicant configuration file based wireless connection'
   Interface=wlp1s0
   Connection=wireless
   Security=wpa-config
   WPAConfigFile='/etc/wpa_supplicant/wpa_supplicant.conf'
   IP=dhcp
   ```

5) ネットワークサービスを開始し、動作することを確認する

   ```
   [root@myhostname ~]# systemctl start systemd-networkd
   [root@myhostname ~]# systemctl start systemd-resolved
   [root@myhostname ~]# ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
   [root@myhostname ~]# netctl start wireless-wpa-config
   [root@myhostname ~]# ping archlinux.org
   ```

6) 接続が確認できたら、起動時にサービスが開始されるようにする。

   ```
   [root@myhostname ~]# systemctl enable systemd-networkd
   [root@myhostname ~]# systemctl enable systemd-resolved
   [root@myhostname ~]# netctl enable wireless-wpa-config
   ```

7) 再起動後、rootでログインし、ネットワークに接続できていることを確認する。

## 基本設定

### ユーザの作成

root以外のユーザを作成し、このユーザでインストール作業を行えるようにしておく。

1) sudoが使えるだけで良いのだが、いずれ必要になるためsudoを含む開発ツールパッケージを導入しておく。

   ```
   [root@myhostname ~]# pacman -S base-devel
   ```

2) ユーザを追加する。

   ```
   [root@myhostname ~]# useradd -m -g users -G wheel username
   [root@myhostname ~]# passwd username
   ```

3) /etc/sudoersを編集し、wheelグループに全コマンドの実行権限を与える。

   ```
   [root@myhostname ~]# visudo
   ---
   %wheel ALL=(ALL) ALL
   ```

これから先はsudoコマンドを使って、ここで作成したユーザでインストール作業を行う。

一旦、ログアウトし、作成したユーザでログインしておく。コマンドプロンプトのusernameは適宜置き換えること。

### 時刻の設定

時刻の設定はArch Linuxのガイドに従う。つまり、ハードウェアクロックはUTCにしておき、Dual Bootの場合はWindows 11もUTCで動作させる。Windowsの設定については省略する。

1) Arch Linuxにntpをインストールする。

   ```
   [username@myhostname ~]$ sudo pacman -S ntp
   ```

2) ネットワーク接続時のみにntpdを実行するように設定する。

   ```
   [username@myhostname ~]$ sudo vi /etc/netctl/wireless-wpa-config
   ---
   (下記を追加する)
   ExecUpPost='/usr/bin/ntpd || true'
   ExecDownPre='killall ntpd || true'
   ```

3) /etc/ntpd.confを修正する

   ```
   [username@myhostname ~]$ sudo vi /etc/ntpd.conf
   ---
   ...
   # Associate to Arch's NTP pool
   # server 0.arch.pool.ntp.org
   # server 1.arch.pool.ntp.org
   # server 2.arch.pool.ntp.org
   # server 3.arch.pool.ntp.org
   server 0.jp.pool.ntp.org
   server 1.jp.pool.ntp.org
   server 2.jp.pool.ntp.org
   server 3.jp.pool.ntp.org
   ```

### TRIMの設定

SSDの速度低下を抑制するためにTRIMの設定を行う。

1) hdparmをインストールする。
   ```
   [username@myhostname ~]$ sudo pacman -S hdparm
   ```

2) SSDがTRIMをサポートしているかどうか確認する。
   ```
   [username@myhostname ~]$ sudo hdparm -I /dev/sda | grep TRIM
   ```

3) SSDがTRIMをサポートしていることが確認できたら、fstrim.timerをsystemdに登録し、一週間ごとにTRIMを実行させるようにする。

   ```
   [username@myhostname ~]$ sudo systemctl enable fstrim.timer
   ```

### CtrlキーとCaps Lockキーの入れ替え

操作性を改善するためにキーボードのCtrlキーとCaps Lockキーを入れ替える。

1) キーマップを格納するディレクトリを作成する

   ```
   [username@myhostname ~]$ sudo mkdir -p /usr/local/share/kbd/keymaps
   ```

2) キーマップを作成する

   ```
   [username@myhostname ~]$ sudo vi /usr/local/share/kbd/keymaps/local.map
   ---
   include "/usr/share/kbd/keymaps/i386/qwerty/jp106.map.gz"
   keycode 29 = Caps_Lock
   keycode 58 = Control
   ```

3) 起動時にキーマップが読み込まれるように設定する

   ```
   [username@myhostname ~]$ sudo vi /etc/vconsole.conf
   ---
   KEYMAP=/usr/local/share/kbd/keymaps/local.map
   ```

## 非GUI環境での日本語表示設定

### フォントのインストール

表示用のフォントにはInconsolataとMigu 1Mフォントを使用する。

この2つはRictyフォントの元になるフォントだが、Rictyを生成して使用するより、合成前の状態で使用したほうが見栄えは良いようだ。

```
[username@myhostname ~]$ pacman -S ttf-inconsolata
[username@myhostname ~]$ pacman -S ttf-migu
```

### kmsconのインストール

コンソールソフトウェアのkmsconをインストールする。AURに開発版が含まれているが、公式パッケージのもののほうが苦労は少ない。

```
[username@myhostname ~]$ sudo pacman -S kmscon
```

### kmsconの設定

kmsconの起動オプションは、/etc/kmscon/kmscon.confに書いておくことができる。フォントの指定もできるが、ここではキーボードの設定だけをを行うことにする。

```
[username@myhostname ~]$ sudo mkdir /etc/kmscon
[username@myhostname ~]$ sudo vi /etc/kmscon/kmscon.conf
---
xkb-layout=jp
xkb-model=jp106
xkb-options=ctrl:swapcaps
```

フォントの設定は、/etc/fonts/conf.d/99-kmscon.confで行う。先にインストールしたInconsolataを英数文字に、Migu 1Mを日本語文字に使用する。

```
[username@myhostname ~]$ sudo vi /etc/fonts/conf.d/99-kmscon.conf
---
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
<match>
        <test name="family"><string>monospace</string></test>
        <edit name="family" mode="prepend" binding="strong">
                <string>Inconsolata</string>
                <string>Migu 1M</string>
        </edit>
</match>
</fontconfig>
```

ここで、kmsconを起動し、動作確認をしておく。スーパーユーザの特権で実行する必要がある。

```
[username@myhostname ~]$ sudo kmscon
```

次のようにすれば、起動時にkmsconが立ち上がるようにすることができるが、kmsconからはstartxでXウインドウを起動できないようなので、注意が必要である。

```
[username@myhostname ~]$ sudo rm /etc/systemd/system/getty.target.wants/getty@tty1.service
[username@myhostname ~]$ sudo systemctl enable kmsconvt@tty1.service
```

## 日本語環境の構築

### 準備

ここでインストールするソフトウェアのいくつかはArch Linuxの公式パッケージには含まれておらず、AUR(Arch User Repository)を使用することになる。AURを使うためにはgitが必要になるため、あらかじめインストールしておく。

```
[username@myhostname ~]$ sudo pacman -S git
```

また、作業用のディレクトリを作成しておく。

```
[username@myhostname ~]$ mkdir -p ~/work/aur
```

### Emacsのインストール

Emacsは公式リポジトリに含まれているので、pacmanでインストールする。

```
[username@myhostname ~]$ sudo pacman -S emacs
```

### Mozcのインストール

日本語変換ソフトウェアのMozcは公式リポジトリにfcitx-mozcとして含まれているが、Emacsへの対応がなされていない。AURからmozcパッケージをダウンロードする。

```
[username@myhostname ~]$ cd ~/work/aur
[username@myhostname aur]$ git clone https://aur.archlinux.org/mozc.git
```

Emacsで使用できるようにするためには、PKGBUILDファイルを修正する必要がある。修正後、makepkgを使用してビルドとインストールを行う。

```
[username@myhostname aur]$ cd mozc
[username@myhostname mozc]$ vi PKGBUILD
---
## Maintainer: ponsfoot <....>

### If you will be using mozc.el on Emacs, uncomment below.
_emacs_mozc="yes"
...
---
[username@myhostname mozc]$ makepkg -si
```

### cmigemoのインストール

ローマ字で日本語検索を可能にするcmigemoをビルドするためには、文字コード変換ツールのnkfが必要になるが、自動ではインストールされないようなので、先にインストールしておく。

```
[username@myhostname ~]$ cd ~/work/aur
[username@myhostname aur]$ git clone https://aur.archlinux.org/nkf.git
[username@myhostname aur]$ cd nkf
[username@myhostname nkf]$ makepkg -si
```

cmigemoもAURからダウンロードしてインストールする。

```
[username@myhostname nkf]$ cd ~/work/aur
[username@myhostname aur]$ git clone https://aur.archlinux.org/cmigemo-git.git
[username@myhostname aur]$ cd cmigemo-git
[username@myhostname cmigemo-git]$ makepkg -si
```

## GUI環境のインストール

非GUI環境はマシンへの負荷が低い代わりにユーザができることも限られる。テキスト入力だけの用途であっても、Emacsで変換キー、無変換キーなどの特殊キーが使用できないため、日本語入力の際に若干不便さを感じる。そこでGUI環境を導入するが、できるだけ軽量化するためにディスプレイマネージャにはLightDM、ウインドウマネージャはi3-wmを使用する。

### Xサーバー(Xorg)のインストール

1) Xorgをインストールする。

   ```
   [username@myhostname ~]$ sudo pacman -S xorg-server xorg-xinit
   ```

2) ビデオドライバをインストールする。インストールしなくても動くが、入れておくとバックライトの調整ができない。他にも不具合があるかもしれない。

   ```
   [username@myhostname ~]$ sudo pacman -S xf86-video-intel
   ```

3) 後々、入力デバイスの情報を取得する必要が出てくるため、xorg-xinputをインストールしておく。

   ```
   [username@myhostname ~]$ sudo pacman -S xorg-xinput
   ```

4) 日本語キーボードの設定を行う。

   ```
   [username@myhostname ~]$ sudo localectl --no-convert set-x11-keymap jp jp106 "" ctrl:swapcaps
   ```

5) Emacs用に日本語フォントRictyをインストールする。AURを使用するが、PKGBUILDファイル中に記載されているダウンロード元とシェルスクリプト名を修正する必要がある。

   ```
   [username@myhostname aur]$ git clone https://aur.archlinux.org/ttf-ricty.git
   [username@myhostname aur]$ cd ttf-ricty
   [username@myhostname ttf-ricty]$ sudo pacman -S fontforge
   [username@myhostname ttf-ricty]$ vi PKGBUILD
   ---
   ...
   source=('https://rictyfonts.github.io/files/ricty_generator.sh')
   ...
     chmod +x ./ricty_generator.sh
     ./ricty_generator.sh -dr0 ...
   ...
   ---
   [username@myhostname ttf-ricty]$ makepkg -si
   ```

### ウインドウマネージャ(i3)のインストール

1) i3とi3の標準アプリケーションランチャーであるdmenu、もう一つのアプリケーションランチャーであるrofiをインストールする。また、ターミナルソフトウェア(urxvt)とスクリーンロックの際に使用するxss-lockもここでインストールしておく。

   ```
   [username@myhostname ~]$ sudo pacman -S i3
   [username@myhostname ~]$ sudo pacman -S dmenu
   [username@myhostname ~]$ sudo pacman -S rofi
   [username@myhostname ~]$ sudo pacman -S rxvt-unicode
   [username@myhostname ~]$ sudo pacman -S xss-lock
   ```

2) i3のステータス行の設定を行う。ipv6、Ethernetは不要なので表示しないようにコメントアウトする。

   ```
   [username@myhostname ~]$ sudo vi /etc/i3status.conf
   ---
   # i3status configuration file.
   ...
   # order += "ipv6"
   ...
   # order += "ethernet _first_"
   ...
   ```

3) Xウインドウ起動時にi3が立ち上がるように、.xinitrcを編集する。

   ```
   [username@myhostname ~]$ vi ~/.xinitrc
   ---
   exec i3
   ```

4) Xorgの動作確認を行う。

   ```
   [username@myhostname ~]$ startx
   ```

5) Xorgの起動確認後、ログアウト(i3を終了)する。

### ディスプレイマネージャ(LightDM)のインストール

ディスプレイマネージャにはLightDMを使う。

1) LightDMをインストールする

   ```
   [username@myhostname ~]$ sudo pacman -S lightdm lightdm-gtk-greeter
   ```

2) グラフィックドライバが読み込まれてから起動するようにLightDMを設定する。

   ```
   [username@myhostname ~]$ sudo vi /etc/lightdm/lightdm.conf
   ---
   [LightDM]
   logind-check-graphical=true
   ```

3) LightDMの言語を日本語に設定する。

   ```
   [username@myhostname ~]$ sudo vi /etc/environment
   ---
   LANG=ja_JP.UTF-8
   ```
4) システム起動時にlightdmが開始するようにサービスを登録する。

   ```
   [username@myhostname ~]$ sudo systemctl enable lightdm
   ```

5) システムを再起動し、GUIログイン画面が表示されることを確認する。
