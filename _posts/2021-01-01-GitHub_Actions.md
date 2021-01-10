---
layout: post
title: GitHubのActionでマルチ環境向けのバイナリをビルドして配布する(Rust)
category: blog
tags: rust github git calculator test
---

CLI（command line interface）ツールはRustでも力を入れてりるターゲット。RustはLLVMをバックエンドとしているし、ライブラリも抽象化されている。GUIを扱わない範囲ではWindows/Linux/Macを対象とした移植性があるCLIツールを書きやすい。さらにGitHubではActionを用いたビルドファーム（テストも）がOSSでは利用可能だ。ソースからビルドではなく、多環境向けのバイナリをGitHub Actionsでビルドしてバイナリ配布するための設定について述べる。

2020冬シーズンの篭もりプロジェクトとして`rc`というコマンドラインで動作する関数電卓を作成した。リポジトリは[https://github.com/nkon/rc-rs](https://github.com/nkon/rc-rs)に、設計ノートは[https://github.com/nkon/rc-rs/blob/master/NOTE.md](https://github.com/nkon/rc-rs/blob/master/NOTE.md)にて公開している。この記事は整理と閲覧性向上のために、[設計ノート](https://github.com/nkon/rc-rs/blob/master/NOTE.md)からGitHub Actionsに関して抜き出してまとめたものだ。


[`.github/workflows/rust.yml`](https://github.com/nkon/rc-rs/blob/master/.github/workflows/rust.yml)にアクションを書いておけば、指定したトリガ（`push`をトリガとすることが多い）に対して、CIアクションが走る。それは、ビルドファームでの配布用バイナリビルド、テスト、アクションをトリガとしたGitHubのReleaseページへの追加、などが含まれる。

Windows/Linux/Mac用の配布バイナリを生成するためにGitHub Actionsを使ってみた。Rustはクロスビルドの環境が整っているので`ubuntu-latest`でもWindowsバイナリを生成可能だがテストのこともあるので`windows-latest`でセルフビルドしてみた。

事前学習として`GitHub/Actions`と`actions`、`actions-rs`を見ておく必要がある。[`GitHub/Actions`](https://docs.github.com/ja/free-pro-team@latest/actions)はGitHubで何ができるのかとういうこｔ、[`actions`](https://github.com/actions)では汎用的なアクションを提供しているし[`action-rs`](https://github.com/actions-rs)ではRustに特化したActionsを提供している。それらのActionsはYAMLの中で参照すればインポートできる。

## 設定

GitHubのActionsページに行って[New Actions]ボタンを押せば、主要開発言語であるRustを判別して、適切な初期Actionを設定してくれる。以降はそれの編集について。

具体的な実装については[https://github.com/nkon/rc-rs/blob/master/.github/workflows/rust.yml](https://github.com/nkon/rc-rs/blob/master/.github/workflows/rust.yml)がある。以下はそれの解説。

## トリガパート

```yaml
name: Rust

on:
  push:
    tags:
      - 'v*'

env:
  CARGO_TERM_COLOR: always
```

* `name` ⇒好きに名前を付けてよい。
* `on:`→`push:`→`tags:`→`v*`⇒ `push`時に`tags`が付いていたら発動。VS-Codeの場合は`push`ではtagはpushされないので、コマンドラインでpushするか(`git push --tags`)、コマンドパレット(F1)で`Git Push Tags`を選択する。

`CARGO_TERM_COLOR:` はコマンド応答をカラフルに行う工夫。


## Build

```yaml
jobs:
  build:

    strategy:
      matrix:
        target:
          - x86_64-unknown-linux-musl
          - x86_64-pc-windows-msvc
          - x86_64-apple-darwin
        include:
          - target: x86_64-unknown-linux-musl
            os: ubuntu-latest
          - target: x86_64-pc-windows-msvc
            os: windows-latest
          - target: x86_64-apple-darwin
            os: macos-latest

    runs-on: ${{ matrix.os }}

    steps:
      - name: Setup code
        uses: actions/checkout@v2

      - name: Install musl tools
        if : matrix.target == 'x86_64-unknown-linux-musl'
        run: |
          sudo apt install -qq -y musl-tools --no-install-recommends
      
      - name: Setup Rust toolchain
        uses: actions-rs/toolchain@v1
        with:
          profile: minimal
          toolchain: stable
          target: ${{ matrix.target }}
          override: true

      - name: test
        uses: actions-rs/cargo@v1
        with:
          command: test

      - name: Build
        uses: actions-rs/cargo@v1
        with:
          command: build
          args: --release --target=${{ matrix.target }}

      - name: Package for linux-musl
        if: matrix.target == 'x86_64-unknown-linux-musl'
        run: |
          zip --junk-paths rc-${{ matrix.target }} target/${{ matrix.target }}/release/rc

      - name: Package for windows
        if: matrix.target == 'x86_64-pc-windows-msvc'
        run: |
          powershell Compress-Archive -Path target/${{ matrix.target }}/release/rc.exe -DestinationPath rc-${{ matrix.target }}.zip

      - name: Package for macOS
        if: matrix.target == 'x86_64-apple-darwin'
        run: |
          zip --junk-paths rc-${{ matrix.target }} target/${{ matrix.target }}/release/rc

      - uses: actions/upload-artifact@v2
        with:
          name: build-${{ matrix.target }}
          path: rc-${{ matrix.target }}.zip
```

`strategy:`、`matrix:`でターゲットを複数定義する。今回は`x86_64-unknown-linux-musl`、`x86_64-pc-windows-msvc`、`x86_64-apple-darwin`の2種類。それぞれのターゲットに応じてビルドOSを設定する。

`runs-os`でビルド用のOSを起動。

わかりやすいように`steps`では`name`を付けている。muslツールは標準ではないので`apt`で追加のセットアップを行う。Rustのツールチェインのセットアップは`acsions-rs`に用意されているものを使う。その上で、testとbuildを実施。クレートをキャッシュする方法もあるようだが、今回は用いていない。私が試した時は、浮動小数点の演算精度のせいで特定のターゲットでテストがFailしたりした。

Linux版はMUSLによるスタティックリンクされたバイナリ、Windows版でも[`.cargo/config`](https://github.com/nkon/rc-rs/blob/master/.cargo/config)の記述に従い、スタティックリンクされたバイナリが作成される。


その後zipでパッケージを作成する。しかし、`windows-latest`ではzipコマンドが用意されていない。powershellの内蔵コマンドでzipパッケージを作成する。
パッケージは次のジョブで利用されるのでアーティファクトとしてアップロードしておく。


## Create-Release、

```yaml
  create-release:
    needs: [build]
    runs-on: ubuntu-latest
    steps:
      - id: create-release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
          draft: false
          prerelease: true
      - run: |
          echo '${{ steps.create-release.outputs.upload_url }}' > release_upload_url.txt
      - uses: actions/upload-artifact@v1
        with:
          name: create-release
          path: release_upload_url.txt
```

YAMLなのでインデントレベルが重要。`create-release:`のインデントレベルは`build:`と同じにする。

`build`がすべて（3つとも）完了したら`create-release:`がトリガされる。

`GITHUB_TOKEN`はこのように書いておけば勝手に渡してくれる。GitHubにプッシュされたタグからリリース名を作って`actions/create-release`でリリースを作成する。そうすれば、GitHubはプロジェクトごとに[リリースページ](https://github.com/nkon/rc-rs/releases)を持っており、そこに項目が作成される。

## Release

```yaml
  upload-release:
    strategy:
      matrix:
        target:
          - x86_64-unknown-linux-musl
          - x86_64-pc-windows-msvc
          - x86_64-apple-darwin
    needs: [create-release]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v1
        with:
          name: create-release
      - id: upload-url
        run: |
          echo "::set-output name=url::$(cat create-release/release_upload_url.txt)"
      - uses: actions/download-artifact@v1
        with:
          name: build-${{ matrix.target }}
      - uses: actions/upload-release-asset@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          upload_url: ${{ steps.upload-url.outputs.url }}
          asset_path: ./build-${{ matrix.target }}/rc-${{ matrix.target }}.zip
          asset_name: rc-${{ matrix.target }}.zip
          asset_content_type: application/zip
```

作成したリリースに、zipパッケージをアップロードする。`actions`の仕様だと思うがソースコードの`zip`と`tar.gz`もアップロードされる。

### 余談

いろいろ試している時に、tagを付けずにリリースしていたらGitHubのリポジトリが壊れてえらいことになった。

詳細は忘れてしまったが、`refs/tags/refs/heads/master`というブランチができていた。タグをつけると、`refs/tags/v0.1.0`のようなブランチができるのだが、タグを指定せずにタグをつけたら、`master`の正式名である`refs/heads/master`がタグとして認識されて、へんなブランチができたのだろう。そのせいでリモートに`master`をpushする時に複数のターゲットがマッチするのでpushできない、というような内容だったと思う。

`git ls-remote`で問題のタグのフル名を特定し、そこに`git push origin :refs/tags/refs/heads/master`というpushをすることでそれを削除して復活した。

