---
layout: post
title: Ubuntu 20.04 LTS 日本語 Remix インストールメモ
category: blog
tags: ubuntu linux
---

2022年のこの時期に、Ubuntu 20.04 LTS日本語Remixのインストールメモ。本当は22.04を入れようとしたのだが、Snapがらみのバグがまだまだ多そうなので、20.04にした。本記事はインストールメモであるが、長年、Linux環境を乗り継いてきた構成ノウハウについての解説でもある。


## PCのディスク構成

このPCは2012年に組んだもので、かなり古い構成。しかし、ドライブやパーテションの切り方はその時点でのノウハウの総まとめになっていて、長年使えるように工夫されている。実際、部品を入れ替えつつ、10年にわたって連続稼働している。しかし、そろそろ新しいマシンが欲しいものだ。

特徴としては、`/dev/sda5`と`/dev/sda6`の2つのLinux OS用パーテションがあり、交互にインストールしていくことにより、1つ前のバージョンをいつでも起動することができる。

現状、`/dev/sda5`に旧Ubuntuが入っているので`/dev/sda6`にUbuntu 22.04をインストールする。

Ubuntu18.04以降、起動のために`/boot/efi`パーテションが要求される。`/dev/sda3`はswapに使っていたが、swapを使わずにそこをEFIパーテションに割り当てる。`/dev/sda5`に入っている旧Ubuntuでも`swap off`して`/etc/fstab`からswapパーテションの記述を削除して、EFIパーテションがswapとして使われないようにしておく。

|Disk    |Partition|Size  |Description                 |
|--------|---------|------|----------------------------|
|/dev/sda|         |256GB |SSD STAT接続。OSが入る      |
|        |/dev/sda1|100MiB|NTFS Windows起動パーテション|
|        |/dev/sda2|74GiB |NTFS Windows OSパーテション。今はWindows 10が入っている。1年に1回ぐらい起動するかも|
|        |/dev/sda3|34GiB |Linux swap                  |
|        |/dev/sda4|130GiB|拡張パーテション。sda5,sda6を格納|
|        |/dev/sda5| 65GiB| linux 1                    |
|        |/dev/sda6| 65GiB| linux 2                    |
|/dev/sdb|         |8TB   |HDD STAT接続。ホームディレクトリが入る。/home/ にマウントされる|
|/dev/sdc|         |8TB   |HDD STAT接続。バックアップ  |

OSはSSDに入っているがデータはHDD接続。ホームディレクトリ用とバックアップ用になっている。

`/home/`以下のツリー構成は次のようになっている。

```
/home/
└data/
└myuser/
└myuser-vine26/
└myuser-vine31/
└myuser-vine41/
└myuser-vine51/
└myuser-ubuntu1604/
└myuser-ubuntu1804/
└myuser-ubuntu2004/
```

今回の場合は`/home/myuser-ubuntu2004`が`myuser`でログインしたときのホームディレクトリ（`${HOME}`）になる。しかし、人間が作成したコンテンツ類はすべて`/home/myuser`以下に入っており、`${HOME}`からシンボリックリンクが張られている。写真や動画、ISOイメージなどの大きなファイルは`/home/data`以下に入っており、これらも`${HOME}`からシンボリックリンクとなっている。`/home/data`は、バックアップから引き継ぎながら使いまわしており、黒歴史も含めて、継続使用している。

こうすうることで、前のOSでログインしたら前のホームディレクトリがそのまま使える。データはどのOSにログインしても共用できる。ホームディレクトリは設定ファイルなどが散らばりがちだが、それらはOS固有のホームディレクトリに保存され、交じり合わない。また、太古の設定ファイルを掘り返すことも可能。

## Ubuntu 22.04 LTS 日本語 Remix のインストール

* インストールメディアを作成して、そこから起動。上述のように`/dev/sda6`にインストールする。ここに入っていた、2つ前のLinuxは全消去。
* 上述のようにEFI用のパーテションが要求されるので`/dev/sda3`を指定。


## 絶対やった方がいいオススメの設定


### フォルダ名を英語にする

「ドキュメント」などのフォルダ名を「Documents」とする。

端末から次のコマンドを実行。
```
LANG=C xdg-user-dirs-gtk-update
```
ウィンドウが表示される→Update Namesボタンをクリック。ログアウトして再ログインすると、問い合わせウィンドウが表示される。「次回から表示しない」にチェックすると、英語名のまま継続利用できる。


### アプリウィンドウを最大化しない

デフォルトだと、アプリ起動時にウィンドウが最大化状態で起動する。次のコマンドで設定を変更して、この機能を無効化する。

```
gsettings set org.gnome.mutter auto-maximize false

```
元に戻すコマンド。
```
gsettings reset org.gnome.mutter auto-maximize
```

### synapticのインストール

GUIでパッケージの検索と更新ができるのは便利。

```
sudo apt install synaptic
```

### キーバインドの設定

日本語キーボードを使っている場合。

* 「A」の横は「CTRL」
* 「半角/全角」と「Esc」の入れ替え

`/etc/default/keyboard` に `XKBOPTIONS=*ctrl:swapcaps,japan:jztg_escape*`を追記

* 「変換」でIM ON、「無変換」でIM OFF。トグルよりも、押すと必ず入力モードが確定する。
 
IMEの変換キー設定で次を設定

* 変換中:Muhenkan=IME無効化
* 変換前入力中:Muhenkan=IME無効化
* 直接入力:Henkan=IME有効化


### キーリングの削除

ときどき、キーリングのパスワードを要求される。設定した覚えがないので毎回エラーになる。
次のページにしたがって「Login」パスワードを消去

[https://help.media.hiroshima-u.ac.jp/index.php?solution_id=1039](https://help.media.hiroshima-u.ac.jp/index.php?solution_id=1039)


## デスクトップ環境

旧環境ではLXDEを使っていた。

今回、Ubuntu Mateを試してみようと思ったが、LightDMからログインしてもデスクトップが起動しない。

結局Xfceをデスクトップ環境として使うことにした。

### Firefox

Firefoxのブックマークとパスワードを、旧環境でエクスポートして、新環境でインポートする。


### ソフトウエアのインストールと設定

一部のソフトはsnap経由になっている。

新し目で試しているものとして、

* 端末ソフトはalacrittyを使ってみる。GPUアクセラレーションが効いて、画面がすっきりしていて綺麗。設定がFreeDesktop寄り（`~/.config/alacritty/alacritty.yml`）なのも現代的。
* シェルは相変わらずzsh + Oh-my-Zsh。
* ファイルマネージャは、相変わらずXfce標準のThunar。ディレクトリとファイルをまとめてソートできるところが気に入っている。


## サーバ環境の設定

