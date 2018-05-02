
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

![990131154018709504](/images/990131154018709504.png)

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

