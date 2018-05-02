
class: center, middle

# Rust から学ぶ Good Practice

---

# Disclaimer

* 社内勉強会用に考えていたネタだが、機会がなくなったので公開する
* 内容は初心者向け
* ツッコミどころ多数
* 後半、力尽きている

---

# なぜいろいろなプログラミング言語があるのか？

* プログラム言語は、最終的にマシン語となり実行される
* アセンブラで書けば、全てのプログラミングに求められる機能は実行できる
* CでLinuxカーネルさえも書ける

---

# プログラミング言語の機能

よくあるパターンを簡潔に書く

* 記述量削減→見通し向上
* 人為的バグの排除
* ライブラリとして実現できるか？ 言語機能として必要か？
    + 言語を拡張できる言語機能(Lisp等)
    + 関数呼び出しで実現できない機能

---

# プログラミング言語の機能

危険なことをできないように

* (例)ポインタの排除
* プログラマが注意すればいいのか？ 言語に縛られると自由度が下がる
    + だれもがハッカーではない

---

# プログラミング言語の機能

新しい世代の言語は、古い言語の課題・反省をもとにしている

1. アセンブラ, Fortran
2. C, C++, Pascal: 構造化、オブジェクト指向
3. Java, Perl, Ruby: 動的型付け、GC
4. Haskell, JavaScript: 関数型
5. Go, Swift, Rust: コンカレンシー

新しい言語の機能・考え方を古い言語を使う時のプラクテスとして取り込む

---

# 今回の目的

Rust 言語の機能から C などでの開発に、規約として取り込めそうなものを考える。

Rustの解説として [TRPL: The Rust Programing Language 2nd(和訳)](https://y-yu.github.io/trpl-2nd-pdf/book.pdf) を使う。

---

# Rust とは

安全性、速度、並行性の3つのゴールにフォーカスしたシステムプログラミング言語

* ガベージコレクタなしにこれらのゴールを実現
* 他の言語への埋め込み、要求された空間や時間内での動作、 デバイスドライバやオペレーティングシステムのような低レベルなコードなど他の言語が苦手とする多数のユースケースを得意
* 全てのデータ競合を排除しつつも実行時オーバーヘッドのないコンパイル時の安全性検査を多数持つ
* 高級言語のような抽象化も含めた「ゼロコスト抽象化」も目標とする
* そうでありつつもなお低級言語のような精密な制御も許す
* Cなどの他言語とのバインディング

[TRPL 1.6](https://rust-lang-ja.github.io/the-rust-programming-language-ja/1.6/book/)より

例えば、メモリ管理やデータ表現、非同期などの低レベルな詳細を扱う「システムレベル」の作業を例にとってください。伝統的にこの範囲のプログラミングは、難解と見なされ、有名な落とし穴を回避することを学ぶのに必要な年月を捧げた選ばれし者にだけアクセス可能です。鍛錬を積んだ者でさえコードが脅威やクラッシュ、頽廃に開かれないように、注意深く行うのです。Rustは、古い落とし穴を排除し、その過程で助けになる親しみ深い洗練された一連のツールを提供することで、これらの障壁を破壊します。

[TRPL 2](https://y-yu.github.io/trpl-2nd-pdf/book.pdf)より

---

# Rustの主な機能

* 言語
    + 不変性
    + 所有権
    + トレイとを使ったオブジェクト指向
    + エラーの返し方
    + 名前空間
* 規約
    + 命名基準
    + ドキュメントの書き方、項目
* 標準ツール
    + 外部ライブラリの取り込み、バージョン管理
    + cargo doc
    + cargo install-update, rustup
    + ツールバージョン管理
    + ユニットテスト、結合テスト、Docテスト
    + プロジェクトディレクトリ
* リポジトリ

---

# Install, Update

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p8

.left-column[
### Install
```
$ curl https://sh.rustup.rs -sSf | sh
```
### Update
```
$ rustup update
```
]

.right-column[

* OSの機能を使わず、自前のインストール、アップデート機能を持つ→良し悪し
* `rustfmt`という整形ツールがオフィシャルから提供→スタイルの統一、not宗教戦争

]

---

# Project

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p14

.left-column[
### プロジェクトの作成
```
$ cargo new hello_cargo --bin
```
### プロジェクトのビルド
```
$ cd hello_cargo
$ cargo build
$ tree -a
.
├── .git      # 略
├── .gitignore
├── Cargo.lock
├── Cargo.toml
├── src
│   └── main.rs
└── target
    └── debug
        ├── hello_cargo
        略
```
]
.right-column[

* Gitのリポジトリ、`.gitignore`の設定もなされる→VCSはGitがデファクト
* ソースは`src/`、生成物は`target/`→ビルドは別ディレクトリ
* 自前のビルドツールを持ち、`make`を使わない→`make`は闇、習得が難しい

]

---

# 変数

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p14

.left-column[
```rust
let mut guess = String::new();
```
]
.right-column[

* 変数は、デフォルトで不変→Cなら`const`を付ける
* 型推定→Cでも型をきちんと区別して使う

]


---

# IOエラーの取り扱い

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p22

.left-column[
```rust
    io::stdin().read_line(&mut guess).expect("Failed to read line");
```
]

.right-column[
`Result<T,E>`を返す：Cでは無い概念

* エラー処理の強要(`Result`から`<T>`をとり出さなければならない)→Cではエラー返値を無視できるが、コードレビューでカバー
* 標準のエラー処理(`expect`)を用意して手間を省く→エラー処理は個別で書かずにライブラリ化して使い回す
* 呼び出し側でメモリ確保(`&mut guess`)→Cでもそうする。内部で確保したメモリを返すとリークする

]

---

# 外部ライブラリ

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p25

.left-column[

`Cargo.toml`
```
[dependencies]
rand = "0.3.14"
```

`main.rs`
```rust
extern crate rand;
use rand::Rng;
let secret_number = rand::thread_rng().gen_range(1, 101);
```

]

.right-column[

* ライブラリのバージョンは[SemVer](https://semver.org/lang/ja/)を使う→互換性を判定するには必要にして十分
    + SemVerはライブラリのバージョンには適するが、製品のバージョンに適するかどうかは個別に検討
* `cargo`が creats.io からダウンロードしてビルドする→パッケージマネージャも兼ねる
* `Cargo.lock`でバージョンをロックして、`cargo update`でSemVer互換性を考慮してライブラリをバージョンアップする
* `extern crate rand;`で外部パッケージを宣言し、`use rand::Rng;`で名前空間を導入し、`rand::thread_rng().gen_range(1, 101);`でライブラリ関数を使う→名前空間はCには無い。APIの命名で工夫するしかない
* `cargo doc --open`でライブラリドキュメントが見れる→コードとドキュメントの一体化、乖離の予防

]

---

# `match`での比較

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p29

.left-column[
```rust
match guess.cmp(&secret_number) {
    Ordering::Less => println!("Too small!"),
    Ordering::Greater => println!("Too big!"),
    Ordering::Equal => println!("You win!"),
}
```
]

.right-column[

* 漏れなく比較しなければコンパイルエラー→MISRA-Cなどのツールでチェック

]


---

# グローバル変数

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p41

* 変数：ローカルスコープ、デフォルトで immutable
* 定数：ローカルまたはグローバルスコープ(`'static` なライフタイム)
* グローバル変数：Rustでは`unsafe`として扱われる→C でも基本的にグローバル変数は使わない。どこかで勝手に変更されるかもしれない変数はとても危険！


---

# 整数型

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p43


.left-column[

|大きさ|符号付き|符号なし|
|------|--------|--------|
|8bit  | i8     | u8     |
|16bit | i16    | u16    |
|32bit | i32    | u32    |
|64bit | i64    | u64    |
|arch  | isize  | usize  |

]

.right-column[

* 基本的に整数の符号とサイズを意識する→Cでは`<stdint.h>`
* 論理型は`<stdbool.h>`
* 特殊型を活用する、なんでも `int`にせずに、コンパイラの型チェック機能を活かす

]

---

# 関数

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p49


.left-column[

```rust
fn main() {
    println!("Hello, world!");
    another_function();
}

fn another_function() {
    println!("Another function.");
}

fn five() {
    5
}
```

]

.right-column[

* 関数宣言のキーワードは`fn`→ (明言無いが)予約後は短縮形、(ユーザ)識別子は短縮しない、で衝突を避ける思想 → 短縮後は使わないようにしよう(システムレベルで共通認識が無いとわかりにくい)
* 関数名は snake_case ← [規約](https://sinkuu.github.io/api-guidelines/naming.html)あり
    + 他の項目も超参考になる
* ブロック(枝)の最後の式の値が返り値になる→C でも `return`文よりも後の式を許さない(コンパイラ警告？ )

]


---

# 所有権

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p66

* とても複雑で重要。Cの感覚で書くと、ほぼ借用チェッカーに引っかかる。
* Cでも、参照するときに、immutable な参照か mutable な参照かを意識したコードを書こう。← Rustでは、mutable な参照を渡すと「所有権」を手放す。
* 値を変更するのは誰か？ 意図しない変更、辿れないポインタ、有効でないメモリ領域への参照を防ぐ。

---

# Null安全

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p109

* [null安全でない言語は、もはやレガシー言語だ](https://qiita.com/koher/items/e4835bd429b88809ab33)
* Cでは言語レベルで保証してくれないので、コードレビューなどで確認する。
    + 人力では限界がある。
    + test, `assert` も活用しよう。


---

# モジュール

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p131

.left-column[

```rust
pub mod client;
mod network;
```

]

.right-column[

* モジュール内は、デフォルトで不可視、`pub`を付けたものだけ公開 → C でも `static`キーワードを活用しよう

]

---

# Collection

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p140

.left-column[

[https://twitter.com/yoh2_sdj/status/990131154018709504](https://twitter.com/yoh2_sdj/status/990131154018709504)

![990131154018709504](images/990131154018709504.png)

]

.right-column[
* 何でもかんでも、配列＋`for`ではない。
* Collection ライブラリを作成し、使い回して活用しよう。
    + `Queue`や`List`はあるが
    + `Array`、`Set`などもライブラリ化して独自実装はなるべくしない
        - 例) 合計が欲しい時も、for でまわして合計するのではなく、sum などのメソッドを使う
    + `Hash`などの有用なコレクションの利用を促進する
    + 信頼できるコードを活用し、しょうもないバグの混入を防ぐ

]

---

# エラー処理

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p172

.left-column[
panic! すべき時と Result を返すべき時はどう決定すればいいのでしょうか? コードがパニックしたら、回復する手段はありません。回復する可能性のある手段の有る無しに関わらず、どんなエラー場面でも panic! を呼ぶことはできますが、そうすると、呼び出す側のコードの立場に立ってこの場面は回復不能だという決定を下すことになります。 Result 値を返す決定をすると、決断を下すのではなく、呼び出し側に選択肢を与えることになります。呼び出し側は、場面に合わせて回復を試みることを決定したり、この場合の Err 値は回復不能と断定して、 panic! を呼び出し、回復可能だったエラーを回復不能に変換することもできます。
]

.right-column[

* *ユーザが*回復可能な時：エラーを返す
* コードのバグでエラーなとき(*プログラマが*対処しなければならない時)：assertion fail

]


---

# テスト

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p212

.left-column[

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn it_works() {
        assert_eq!(2 + 2, 4);
    }
}
```

```
$ cargo test
```

* lib クレート生成したら、スケルトンがテストの作成を促す

]

.right-column[

* ユニットテストは、プログラム作成者のマナー
    + 未テストのコードはPush禁止の会社も？
* いかに、ユニットテストをスムーズに実行する仕組みを構築するか
* テストはバグあることを示すことはできるが、バグが無いことを示すことは出来ない
    + 静的検証：assertのような表明を、テスト入力無しに、常に成立することを示す

]

---

# アプリケーション例

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p239

ぜひ `minigrep`の開発を打ち込みながら、辿ってみましょう。


---

# ドキュメンテーションコメント

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p304

.left-column[

```rust
/// Adds one to the number given.
/// 与 え ら れ た 数 値 に 1 を 足 す 。
///
/// # Examples
///
/// ```
/// let five = 5;
/// assert_eq!(6, my_crate::add_one(5));
/// ```
pub fn add_one(x: i32) -> i32 {
    x + 1
}
```

]

.right-column[

* Doxygen などと同様
    + Doxygen は広く使われており、PlantUMLなどの拡張機能が豊富
* Rust の場合は Markdown 形式 → Markdown 形式
* `# Examples`などの標準的なセクションを規約で定義
* Doc test でコメント中のコード例がテストコードとして実行される

]

---

# ヒープの開放

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p333

.left-column[
    ある言語では、プログラマがスマートポインタのインスタンスを使い終わる度にメモリやリソースを解放するコードを呼ばなければなりません。忘れてしまったら、システムは詰め込みすぎになりクラッシュする可能性があります。 Rust では、値がスコープを抜ける度に特定のコードが走るよう指定でき、コンパイラはこのコードを自動的に挿入します。
]

.right-column[

* C は `malloc`したら`free`を呼ぶのはプログラマの責任
* C++ のようにデストラクタぐらいは欲しいものだ
    + 「制限付きC++」をうまく導入できれば良いのだが
* 極力、`malloc`を使わなくていいように工夫するのが自己防衛
    + チャンクとして確保して作業を行い、終わったらチャンクごと捨てる、など

]

---

# 非同期処理

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p359

* この章ではOSサポートがあるThread処理について扱う
* 組み込みでの、ペリフェラルへの非同期アクセスについては、[Japaric氏のBrave new I/O](blog.japaric.io/brave-new-io/)に提案がある

.left-column[
    Go 言語のドキュメンテーションのスローガンにある考えです：メモリを共有することでやり取りするな ; 代わりにやり取りすることでメモリを共有しろ。
メッセージ送信非同期処理を達成するために Rust に存在する1つの主な道具は、チャンネルで、Rust の標準ライブラリが実装を提供しているプログラミング概念です。
(メモリ共有は)異なる所有者を管理する必要があるので、複数の所有権は複雑度を増させます。 Rust の型システムと所有権ルールにより、この管理を正当に行う大きな助けになります。


```rust
let (tx, rx) = mpsc::channel();
tx.send(()).unwrap();
```

]

.right-column[

* RTOSには、共有領域の排他制御のために mutex や、Rustのチャネルの代わりに message_queue があることが多い
* message_queue が使えるときは mutex よりも安全
* queue に入れるデータの所有権や二重開放は(現状は)人間が注意する

```C
osMessageQDef(message_q, 5, uint32_t); // Declare a message queue
osMessageQId (message_q_id);           // Declare an ID for the message queue

message_q_id = osMessageCreate(osMessageQ(message_q), NULL);

uint32_t data = 512;
osMailPut(message_q_id, data, osWaitForever);

osEvent event = osMessageGet(message_q_id, osWaitForever);
```

]


---

# State パターン

[TRPL-2nd](https://y-yu.github.io/trpl-2nd-pdf/book.pdf): p395

.left-column[

* デザインパターンの実装例として、Stete パターンを使ったblogの記事発行が取り上げられている。
* それぞれのSteteに対応する空構造体を定義し、遷移を構造体に対するメソッドとして定義する(次のStateに対応する構造体を返す)。
* オリジナルのGoF本でも、TRPLのように、各々のStateに対応したClassを定義している。

]

.right-column[

* Cで実装するときは、2次元の状態遷移表(列が状態に、行がイベントに対応する)を作成し、それぞれに関数ポインタでアクションを定めることが多いだろうか。
* 得失は「ステートパターンの代償」にまとめられている。
* クラスに情報を持たせる
    + へんな呼び出しをコンパイル時に検出できる
    + 実際に動く枝の分しかリソースを消費しない(遷移が疎だと表にした時に大量の無効エントリーが出る)
* 表に情報を持たせる
    + 無効なイベントは、無視ではなく、assertで落とすべきだ
    + 仕組みが汎用的で、非プログラマの人に説明する文書やCASEツール、フレームワークと馴染みが良い

]

---

# Appendix: Github-pages, jekyll でスライドを表示させる

参考：[https://qiita.com/natsukium/items/fd98511320ec1d98b851](https://qiita.com/natsukium/items/fd98511320ec1d98b851)

ディレクトリ構成
```
username.github.io/
 ├_includes/
 │ └slides/
 │    └my_slide.md
 ├_layouts/
 │   └slide.html
 └_posts/
      └YYYY-MM-DD-my_slide.md
```


---

# Appendix: Github-pages, jekyll でスライドを表示させる

`_includes/slides/my_slide.md`
```
# My Awesome Presentation

---

# Agenda

1. Introduction
2. Deep-dive
3. ...

[NOTE]: Note that you need active internet connection to access remark.js script file

---

# Introduction

Hello world!
```

* Front matter は不要。
* `---`がスライド区切り。
* 使えるアイテムは [https://github.com/gnab/remark/wiki/Using-with-Jekyll](https://github.com/gnab/remark/wiki/Using-with-Jekyll)参照。

---
# Appendix: Github-pages, jekyll でスライドを表示させる

`_layouts/slide.html`
```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>{{ page.title | strip_html }}</title>
    <style>
      @import url(https://fonts.googleapis.com/css?family=Yanone+Kaffeesatz);
      @import url(https://fonts.googleapis.com/css?family=Droid+Serif:400,700,400italic);
      @import url(https://fonts.googleapis.com/css?family=Ubuntu+Mono:400,700,400italic);

      body { font-family: 'Droid Serif'; }
      h1, h2, h3 {
        font-family: 'Yanone Kaffeesatz';
        font-weight: normal;
      }
      .remark-code, .remark-inline-code { font-family: 'Ubuntu Mono'; }
    </style>
  </head>
  <body>
    <textarea id="source">
{% include slides/{{ page.slide }} %}
    </textarea>
    <script src="https://remarkjs.com/downloads/remark-latest.min.js">
    </script>
    <script type="text/javascript">
      var slideshow = remark.create();
    </script>
  </body>
</html>
```

* remarkjs を読み込む。
* `<textarea id="source">`の中に Markdown を書くと、remarkjs がスライドにレンダリングする。
* そこに、Liquid 構文で、スライドの元ネタを読み込む。

---
# Appendix: Github-pages, jekyll でスライドを表示させる

`posts/YYYY-MM-DD-my_slide.md`
```
---
layout: slide
title: Title of the Slide
slide: my_slide.md
---
```

* Front matter のみ。
* `layout` で `_layout/slide.html`を指定する。
* `slide` で `_includes/slides/my_slide.html`を指定する→`slide.html`の中の`{% include slides/{{ page.slide }} %}`で読み込まれる。
