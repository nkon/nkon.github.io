---
layout: post
title: GitHub でスライドを作る
category: blog
tags: github jekyll
---

この Blog は GitHub Pages と、その上でホスティングされている Jekyll を使って作成されている。その仕組みの延長で、スライドを作成し公開する方法についてまとめる。

# 前提

* GitHub Pages を使っている。
* (ローカルでビルドしてHTMLをアップするのではなく)ホスティングされている Jekyll を使っている。この場合、ホスティング側にインストールされている拡張しか使えない。

HTMLでスライドを作るためのライブラリは多数あるが、情報量が多く、Markdown でさくさく書ける Remark.js を用いることにする。

主に参考にしたのは[こちら](https://qiita.com/natsukium/items/fd98511320ec1d98b851)。

理解を促進するために、ここで使われている要素を整理しよう。

## GitHub Pages

GitHub からリポジトリではなくHTMLで書かれたWeb Siteを配信する仕組み。

* User Page: アカウントあたり１つ。`https://github.com/アカウント名/アカウント名.github.io`というリポジトリを作成し、その`master`ブランチが`https://アカウント名.github.io/`というアドレスで公開される。
* Project Page: リポジトリごと。`https://github.com/アカウント名/リポジトリ名`というリポジトリの`/docs/`ディレクトリが`https://アカウント名.github.io/リポジトリ名/`というアドレスで公開される。

ブログエンジンとして Jekyll を使うことも出来るし、HTML を直接アップすることも出来る。

## Jekyll

GitHub Pages で使われている Blog エンジン。Markdown の原稿から、静的なHTMLを生成することを特徴とする。Rubyで動いている。

## liquid

Jekyll で使われているテンプレートエンジン。`{`,`}`で囲まれたマークアップを用いる。

* 変数の参照は `{{ }}`で、コマンドは `{% %}`で行う。
* テンプレートは `/_layouts/`に入っている。

## Kramdown

Markdown のレンダリングエンジン。

`_config.yml`の中に次のようなセクションがあり、Kramdown の詳細設定が出来る。

```
# Jekyll 3 now only supports Kramdown for Markdown
kramdown:
  # Use GitHub flavored markdown, including triple backtick fenced code blocks
  input: GFM
  # Jekyll 3 and GitHub Pages now only support rouge for syntax highlighting
  syntax_highlighter: rouge
  syntax_highlighter_opts:
    # Use existing pygments syntax highlighting css
    css_class: 'highlight'
```

## rouge

Markdownの ` ``` ` で囲まれたコード部のシンタックスハイライトする。

上の `syntax_highlighter: rouge`によって、kramdown から呼ばれる。


# Remark.js

スライド作成のためのツールとして、Remark.js を使う。処理の流れは次の通り。

* `_posts/`の下に Blog 記事の原稿となる Markdown(Front Matter 付き)を作成する。
* スライド原稿となるエントリーは、Front Matter に `layout: slide`として、`/_layouts/slide.html`をテンプレートとする。
* `/_layouts/slide.html`の中で、remark.js を読み込む。
* `/_layouts/slide.html`の中で、スライド本体をレンダリングする。

## ディレクトリ

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

## `_posts/YYYY-MM-DD-My_Slide.md`

後述する理由により、このファイルは Front Matter のみ。

* `layout: slide`で、`/_layouts/slide.html`をテンプレートとする。
* `slide: `変数で本体となる Markdown ファイルを指定し、Liquid の機能を使って読み込ませる。

```
---
layout: slide
title: Title of the slides
slide: my_slide.md
---
```

## `_layouts/slide.html`

```
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
      .left-column {
        width: 50%;
        float: left;
      }
      .right-column {
        width: 50%;
        float: right;
      }

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
* remark.js を読み込む。
* `remark.create();`で、`<textarea id="source">`のDOM要素が、Markdown → スライドにレンダリングされる。
* `<textarea>`の中に`{% include {{ }} %}`を使って `slide: `で指定した Markdown を読み込む。
* `.left-column{}`などのCSSを定義しておけば、スライドMarkdown で `.left-column[]`として使える。

## スライド本文

* markdown で書く。
* Front Matter は不要。
* `---`がページ区切り。
* `_layouts/slide.html`の`{% include`の記述と対応させて、`_includes/slides`の下に置く。

スライド本文を別ファイルにしておかなければ、`---`が`<HR>`として水平線と解釈されてしまうようだ。

これらを GitHub に Push すれば、スライドが作成される。

例→[https://nkon.github.io/Learning_from_Rust/#1](https://nkon.github.io/Learning_from_Rust/#1)

原稿→[https://github.com/nkon/nkon.github.io/blob/master/_includes/slides/Learning_from_Rust_slide.md](https://github.com/nkon/nkon.github.io/blob/master/_includes/slides/Learning_from_Rust_slide.md)

