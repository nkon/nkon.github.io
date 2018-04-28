
# Rust から学ぶ Good Practice

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

# hohoho

---

# hohoho


---

# Appendix: Github-pages, jekyll でスライドを表示させる

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

`_includes/slides/my.md`
```

```

---
# Appendix: Github-pages, jekyll でスライドを表示させる

`_layouts/slide.html`
```

```

---
# Appendix: Github-pages, jekyll でスライドを表示させる

`posts/YYYY-MM-DD-my_slide.md`
```

```

