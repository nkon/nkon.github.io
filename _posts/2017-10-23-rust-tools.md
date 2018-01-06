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
    それ以外の方法として、そのディレクトリに`rust-toolchain`というファイルを置いて、`nightly`と書いておいても nightly ツールチェインを使える。
* `rustup target add aarch64-unknown-linux-gnu`: 標準でサポートされているクロスコンパイルツールチェインをインストールする。サポートされているツールチェインは `rustup target list`で表示される。`gnu`と対比して`musl`というキーワードが出てくる。Linux上で動く、静的リンクに最適化したライブラリだ。気が合うのか、rustでは熱心にサポートされている。see http://www.musl-libc.org/ 。

## cargo

標準のビルドツール兼パッケージ管理ツール。

### ビルド系
* `cargo new hogehoge --bin`: hogehogeというbin crate を作成する。デフォルトで`main`が用意されているので`cargo run`でHelloアプリが走る。また、デフォルトでGitリポジトリも設定されている。GitHub.comにアップするときは`git remote add origin https://github.com/your-name/project-name.git`とリモートに`origin`を追加する。
* `cargo new hogehoge`: hogehogeというlib crateを作成する。デフォルトでテストが用意されているので、`cargo test`でテストが走る。
* `cargo build`:ビルドする。結果は`target/`ディレクトリに生成される。
* `cargo clean`:コンパイラ生成物を消去する。
* `cargo run`:binクレートをビルドし実行する。
* `cargo update`:今のクレートが依存しているパッケージを、互換性がある範囲で更新する。
* `cargo test`:テストする。`#[test]`属性が付いているユニットテスト関数、`tests/`ディレクトリにある結合テスト、コメント中に記載されている doc-test が走る。
* `cargo doc`:`target/doc/`ディレクトリにドキュメントを生成する。`use`しているクレートのドキュメントも生成される。
* `cargo install`: そのクレートをビルドして`~/.cargo/bin/`にインストールする。
* `cargo install hogehoge`: hogehoge を crates.io からダウンロードしてインストールする。これを利用して、次で説明する「ツール系」サブコマンドが実現されている。
* `cargo metadata --no-deps`: クレートの構成情報を JSON 形式で出力する。依存関係の情報を更新しないために `--no-deps`をつけると良い。JSON の整形表示や抽出は `jq`が便利。

### ツール系

* `cargo --list`: 現在インストールされている、cargo サブコマンドを表示する。
* `cargo install`: パッケージをインストールする。
* `cargo install-update -a`: インストールされているパッケージをアップデートする。
* `cargo install-update -al`: インストールされているパッケージをアップデートの必要性を表示する。

標準の cargo サブコマンドではなく、`cargo install cargo-update`とやって`cargo-update`パッケージをインストールすると`install-update`サブコマンドが使えるようになる。

これらの追加のサブコマンドは、`~/.cargo/bin/cargo-install-update`のようなパスとファイル名　にインストールされる。

* `cargo tree`: `cargo-tree` サブコマンドをインストールする。クレートの依存関係を表示する。
* `cargo modules`: `cargo-modules` サブコマンドをインストールする。モジュールの依存関係を表示する。

いずれも、nightly コンパイラを要求する。`rustup default nightly`で、nightly 環境を有効にしておく。`.cargo/config`などでクロスターゲットが指定されているディレクトリでインストールしようとすると、そちらの指定が優先されて正しくビルドされないので、通常ディレクトリでインストール作業をしなければならない。

* `cargo expand`: `cargo-expand` サブコマンドをインストールする。マクロを展開して表示する。`cc -E`のようなもの。



#### Windows + PowerShell で Path を追加する

Windows 環境で`cargo install cargo-update`すると`cargo-install-update`のビルドの途中で「`cmake`が無い」というエラーで止まる。chocolaty を使っていれば`cinst cmake`でcmakeがインストールできる。しかし、インストールしただけでは、`cmake.exe`にPathが通っていない。

Windows PowerShell では、環境変数は `env:`ドライブにあるので、環境変数を参照するためには`dir env:`とする。
Pathだけを見やすく参照するためには`$Env:Path.split(";")`としても良い。
追加するためには、文字列の追加のように`$Env:Path += ";C:\Program Files\CMake\bin"`とする。

### zsh 補完

`~/.rustup/toolchains/*/share/zsh/site-functions/_cargo` を取り込むと、zsh で `cargo`の補完が効く。

## xargo

クロスコンパイル用のビルドツール。`cargo`のラッパーであり、特記されること以外は cargo と同じ動作をする。`no_std`環境で動作するので、`core`ライブラリをダウンロードして、指定したターゲットに向けてビルドして、リンクしてくれる。

`cargo install xargo`でインストールされる。

* `xargo build --target thumbv6m-none-eabi`:ターゲットのアーキテクチャを指定してビルドする。


## Visual Studio Code

最近の環境では、`rust-lang`製のRust(rls)拡張をいれておけば、rls(Rust Language Server)のインストールまで自動でやってくれる。

