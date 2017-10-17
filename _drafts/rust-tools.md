---
layout: post
title: rust tools
category: blog
tags: rust
---

rust で標準的に使われるツールや便利なツールの使い方を、毎回検索しなくても済むように備忘録的にまとめる。

## rustup

コンパイラをインストールするツール。OSのパッケージとしてコンバイラをインストールするのではなく、rustupを使ってユーザ毎に環境構築するのが標準。
コンバイラだけでなく関連ツールもインストールできる。複数のバージョン(stable と nightlyなど)を同居させることもできる。

* インストール時には最低限のツールとして`rustup`,`rustc`,`cargo`がインストールされる。
    + UNIX:`curl https://sh.rustup.rs -sSf | sh`。
    + Windows: `rustup-inst.exe`をダウンロードして実行する。MSVC ABI を利用するにはBuild Tools for Visual Studio 2017 のインストールが必要。 
* `rustup show`: インストールしてあるツールチェインを表示する。
* `rustup update`: インストールしてあるツールチェインを最新に更新する。
    ツールチェインは `~/.rustup/toolchains/`以下にインストールされる。
    `rustup`自体は `~/.cargo/bin/`にインストールされているので、ここには PATHを通しておく必要がある。手動で設定するのではなく、`source ~/.cargo/env`するのが推奨の方法だ。
* `rustup self update`: rustup 自身をupdateする。
* `rustup install nightly`: nigytly ツールチェインをインストールする。
* `rustup install nightly-2017-10-17`: 特定の日付の nigytly ツールチェインをインストールする。
* `rustup override set nightly`: そのディレクトリで使用するツールチェインを nightly にする。この情報は `~/.rustup/settings.toml`に保存されている。
    それ以外の方法として、そのディレクトリに`rust-toolchain`ファイルを置いて、`nightly`と書いておいても良い。
* `rustup target add aarch64-unknown-linux-gnu`: 標準でサポートされているクロスコンパイルツールチェインをインストールする。サポートされているツールチェインは `rustup target list`で表示される。`gnu`と対比して`musl`というキーワードが出てくる。Linux上で動く、静的リンクに最適化したライブラリだ。気が合うのか、rustでは熱心にサポートされている。see http://www.musl-libc.org/ 。


## cargo

標準のビルドツール兼パッケージ管理ツール。

### ビルド系

* `cargo new hogehoge --bin`: 
* `cargo new hogehoge`:
* `cargo build`:
* `cargo clean`:
* `cargo run`:
* `cargo update`:
* `cargo test`:
* `cargo doc`:


### ツール系

* `cargo --list`:
*



## xargo

クロスコンパイル用のビルドツール。`cargo`のラッパーである。

## Visual Studio Code

* 

## gdb

