---
layout: post
title: clang-tidy のおすすめの設定
category: blog
tags: c LLVM
---

LLVMプロジェクトが出している静的解析ツール`clang-tidy`のおすすめの設定項目を紹介する。とくに組み込み分野でのCでの使用にフォーカスた設定であり、アプリケーション分野、C++での使用には適さないかもしれないのでご注意いただきたい。

古くからは`CppCkeck`がオープンソースでメジャーな静的解析ツールだったが、正直、どうでもいいようなものが検出されたり、検出してほしいと思うものが検出されなかったりと、検出項目には不満があった。そういった状況で`clang-tidy`を使ってみたら、解析精度、必要なエラーを正しく見つけることができる能力、設定の容易さとフレキシビリティ、CIへの統合の容易さなどで満足がいくものだった。最近は`CppCheck`も`clang-tidy`の結果を統合できるようになってきたが、`clang-tidy`自身と設定ファイルである`.clang-tidy`を組み合わせるのが、最も自由度が高い方法だろう。

VS-codeにもいくつかの拡張が見つかる。現時点で「[Clang-Tidy](https://marketplace.visualstudio.com/items?itemName=notskm.clang-tidy)」はうまく動かなかった。「[Clang Tidy GUI](https://marketplace.visualstudio.com/items?itemName=TimZoet.clangtidygui)」はうまく動いた。

### `clang-format`

同様にLLVMからのツールで、フォーマットを整える`clang-format`もある。

こちらのほうが記事や解説は豊富なようだ。

`clang-tidy`にしても`clang-format`にしても、**コーディングルールに関する議論を、ツールと設定ファイルに落とし込む**ことは、開発効率化のために非常に重要だ。さもないと、同じ議論を繰り返したり、チーム内の感情的対立に至ったりしてしまう。

## clang-tidy の設定ファイル

`.clang-tidy`というファイルをプロジェクトのルートディレクトリに置いておく。

`clang-tidy --dump-config > .clang-tidy`とすればデフォルト設定を吐き出してくれるので、これを編集するのもよいだろう。

ファイルの形式はYAMLなのでわかりやすい。

設定項目は `hogehoge-*`とすればワイルドカードで「設定ON」。`-hogehoge`と頭に`-`をつければ「設定OFF」

設定ファイルの中では `Checks:`という項目に書く。コマンドラインオプションでは`--checks=`というオプションで指定する。

## `compile_commands.json`

"Compile Command Database"とも呼ばれる。CMake由来らしい。

Cの正常なビルドには、`*.c`のソースコードファイルだけでなく、必要なインクルードファイルやら、`#define`の定義やらが必要だ。通常のビルドでは、また特に組み込みなどのクロスビルドでは、これらのオプションは非常に複雑になる。実際にビルドしたときのコマンドラインオプションが、ファイル(`*.c`)ごとに保存される。これらのビルド設定は、当然、ただしく静的解析するのにも必要となる。

`clang-tidy`の場合は`-p`オプションで、`compile_command.json`のディレクトリを指定する(ファイル名は固定)。

## 設定項目

ここから具体的な設定項目について説明していく。基本的には次のようなやり方をお勧めする。

* まず最初に `-*`を指定し、すべてのチェックをOFFにする。
* その次に、`bugprone-*`のように、カテゴリごとにONにする。
* `-bugprone-hogehoge`のように、余計だと判断したものをOFFにする。
* とくに重要だと思うものは`hogehoge-fugaguga`のように、手動でONにする。
* 実際にプロジェクトに適用してみて、細部を調整する。

例外としたい箇所については、ソースコードの該当行に`//NOLINT`のようなコメントを付加することで clang-tidyの出力を抑制することができる。

設定項目にパラメータがあるものは、`CheckOptions`という項目に設定する。

2023年2月時点、普通に `clang-tidy`で検索すると 17.0.0gitのバージョンのページに辿り着く。しかしバイナリで入手可能なのは[15.0.0系列](https://releases.llvm.org/15.0.0/tools/clang/tools/extra/docs/clang-tidy/index.html)が最新。けっこう設定項目が違うので注意が必要。

|項目           |設定|説明                                      |
|---------------|----|----------------------------------------|
|`abseil-`      |OFF |Asbil関係はC++なので組み込みC環境ではOFF  |
|`altera-`      |OFF |組み込みC環境ではOFF  |
|`android-`     |OFF |組み込みC環境ではOFF  |
|`boost-`       |OFF |組み込みC環境ではOFF  |
|`bugprone-*`   |    |`bugprone-*`として基本的にすべて有効にして不要なものをOFFにするのが良い。特に次のものはON推奨|
|`bugprone-easily-swappable-parameters`    |OFF |順序を間違えそうな引数を警告する。あまり役に立たないのでOFF       |
|`bugprone-macro-parentheses`              |ON  |マクロの引数をカッコで囲む。ON推奨                                |
|`bugprone-macro-repeated-side-effects`    |ON  |マクロの引数が複数回評価されることを警告する。                    |
|`bugprone-redundant-branch-condition`     |ON  |冗長な条件式を警告する。                                          |
|`bugprone-reserved-identifier`            |    |`_`で始まる識別子は予約後                                         |
|`bugprone-too-small-loop-variable`        |ON  |ループ変数のビット幅が少ないときに警告する。ついメモリをケチるくせがある人は|
|`bugprone-unused-return-value`            |ON  |返り値を適切にハンドリングすること。`CheckOptions`で対象関数を指定できる    |
|`cert-*`       |    |セキュリティに関わるものなのでON推奨なのだがライブラリが対応していないことがある |
|`cert-err33-c` |ON  |標準ライブラリ関数の返り値をチェックする。ついついサボりがちだが、チェックしておかないと思わぬ事故になる|
|`clang-analyzer-`|    |CLANGの標準の静的解析                          |
|`clang-analyzer-security.insecureAPI.strcpy`|ON  |`strcpy()`は危険なので使ってはならない                          |
|`clang-analyzer-security.insecureAPI.DeprecatedOrUnsafeBufferHandling`|OFF |`snprintf_s()`などを使うように推奨するもの。`_s`系の関数がサポートされていない場合はOFF|
|`misc-static-assert`|OFF |`static_assert()`はC11の機能。C99が一般的な環境では無効|
|`readability-magic-numbers`           |OFF |とくに組み込みでは定数が多く登場する。無理に識別子を与えると読みにくいのでOFFでいいだろう|
|`readability-uppercase-literal-suffix`|ON  |整数のサフィックスの`l`を大文字にする                                                    |
|`darwin-`      |OFF |特定プロジェクトではない|
|`fuchsia-`     |OFF |特定プロジェクトではない|
|`google-`      |OFF |特定プロジェクトではない|
|`hicpp-`       |OFF |特定プロジェクトではない|
|`modernize-`   |OFF |"modern"とは"C++11"のこと、組み込みC環境ではOFF  |
|`mpi-`         |OFF |特定プロジェクトではない|
|`objc-`        |OFF |Obj-Cではない           |
|`openmp-`      |OFF |特定プロジェクトではない|
|`zircon-`      |OFF |特定プロジェクトではない|




## 設定例

とあるプロジェクトでは、こんな感じの`.clang-dity`を使っている。

```YAML
---
Checks: '-*,
    clang-diagnostic-*,
    -clang-diagnostic-format,
    -clang-diagnostic-format-security,
    -clang-diagnostic-implicit-function-declaration,
    clang-analyzer-*,
    clang-analyzer-security.insecureAPI.strcpy,
    -clang-analyzer-security.insecureAPI.DeprecatedOrUnsafeBufferHandling,
    bugprone-*,
    bugprone-macro-parentheses,
    bugprone-too-small-loop-variable,
    -bugprone-narrowing-conversions,
    -bugprone-easily-swappable-parameters,
    -bugprone-reserved-identifier,
    cert-*,
    -cert-dcl03-c,
    -cert-dcl16-c,
    -cert-dcl37-c,
    -cert-dcl51-cpp,
    -cert-err33-c,
    misc-*,
    -misc-unused-parameters,
    performance-*,
    -performance-no-int-to-ptr,
    readability-*,
    readability-duplicate-include,
    readability-else-after-return,
    readability-uppercase-literal-suffix,
    -readability-function-cognitive-complexity,
    -readability-identifier-length,
    -readability-isolate-declaration,
    -readability-magic-numbers,
    '
...
```


## Rustでは

Rust では `cargo clippy`がほぼ同等だろうか。

Rustの場合、これらのものが、ほとんど言語組み込み、または標準ツールで提供されていることが、素晴らしい。設定についても標準が提供されているので、どのオプションを採用するのかという議論の余地が、ほとんどない。


## 独自ルール

`clang-tidy`はC++を使って独自ルールを追加しやすい設計になっているので、自分のプロジェクトにあわせて、独自ルールを定義するのも良いだろう。

