---
layout: post
title: GitHub で Blog を作る
category: blog
tags: github jekyll
---

プログラミング関係のブログを GitHub Pages を使って作成しよう。方法はいくつかある。

1. GitHub のアカウントページ

    https://github.com/ACCOUNT を持っている場合、Github pages という機能によって、https://ACCOUNT.github.io/ という WebSite が使える。

    * https://github.com/ACCOUNT/ACCOUNT.github.io というリポジトリを作成すれば良い。
    * https://ACCOUNT.github.io/ というサイトが作成される。

1. GitHub のプロジェクトページ

    プロジェクト・リポジトリの master ブランチの /doc/ ディレクトリをプロジェクト・ページとして公開する機能がある。

    * まず、master ブランチに doc/ ディレクトリを作成し、commit & push する。
    * https://github.com/ACCOUNT/PROJECT/ のページを開いて、[Settings]→[Options]→[Github Pages] で Source = [master branch doc/ folder] を選択する。

今回は、個人の汎用ブログなので、前者の方法を使う。

GitHub Pages 自体は、単なるサイト公開の機能しか無いので、このままだと HTML 手書き、CSSも自前、更新はcommit & push になる。それでは不便なので、Blog tool を導入する。

1. Initializr

    HTML5 で今風のサイトを作るには HTML5 Boilerplate 及び、生成ツールである Initializer が有名だ。デザインやCSSはなんとかなっても、結局 HTML は手打ちで、ファイル管理も手動だ。

1. jekyll

    GitHub 公式と言って良いのが jekyll(ジキル) だ。これも、使い方が 2通りある。

    * ローカルに入れる

        jekyll 自体は Ruby で書かれた、静的ファイル出力型のブログツールだ。`gem install jekyll` でインストールできる。

    * セットアッブ済みのリポジトリをフォークする
    
        Markdown ファイルを push すると、CI 機能によって、GitHub のサーバで自動的にページ生成が行われる。ローカル環境の構築は不要だ。

1. octopress

    octocat + wordpress という感じの命名。jekyll がページジェネレータであるのに対し、octopress はブログツールである。ただし、ローカルへの環境構築が必要となる。

ここでは、ローカル環境が不要な jekyll を用いてゆこう。

# jekyll-now と GitHub Pages でブログを作る

## 雛形のフォーク

* いきなりだが、https://github.com/barryclark/jekyll-now にアクセスする。
* [fork] ボタンを押して fork する。フォーク先は https://github.com/ACCOUNT/ACCOUNT.github.io だ。
* https://ACCOUNT.github.io/ にアクセスすると、ページが居る。
* https://github.com/ACCOUNT/ACCOUNT.github.io をカスタマイズする。まずはローカルに pull する。

## 初期設定

* `_config.yml` は、最低限、次のところを修正しよう。
   
```
# Name of your site (displayed in the header)
name: Your Name

# Short bio or description (displayed in the header)
description: Web Developer from Somewhere

# URL of your avatar or profile pic (you could use your GitHub profile pic)
avatar: https://raw.githubusercontent.com/barryclark/jekyll-now/master/images/jekyll-logo.png
```

push して、[Setting] の [GitHub Pages] が [Your site is published at...]と緑色になればOK。

## 記事の投稿

* `_post/` ディレクトリに `YYYY-MM-DD-xxxxxxx.md`というファイルを push すれば、記事を投稿したことになる。
* 記事を push したら `https://github.com/ACCOUNT/ACCOUNT.github.io/settings`の Option→GitHub Pages のところで発行状態を確認できる。もしかしたら、ビルドエラーが出てるかも。
* 先頭部分は YAML 形式で Front Matter を書く。もともとあったファイルを参照する。
* それ以外に、`category`, `tags` などが使える。 


```
---
layout: post
title: You're up and running!
---
```

これが、その記事だ。

# カスタマイズ

## Draftの作成

長文の場合は下書きが必要だ。下書きは Publish しないが、GitHubでの管理はしたい。

その場合は `_drafts/`ディレクトリに、日付ナシの *.md ファイルを作成すれば良い。

## コメントの追加

Jykyll は静的HTML作成ツールなのでコメントは外部サービスを使う。Disqus がよく使われるようだ。

disqus のサイトに行って登録する。

最近の Jekyll の `_post/html`には `\{\% include disqus.html \%\}`がすでに設定されている。これは`_includes/disqus.html`を読み込む。
`_includes/discus.html`には、`site.disqus`変数を利用して動作する。

つまり、`_config.yml`の`discus:`が空欄になっているのて、そこに disqus アカウントを設定すれば良い。

## google analytics

同様に、`_config.yml`の `google_analytics`に自分の GAアカウントを設定すれば、`_layout/default.html`で`_includes/analytics.html`を読み込んでいるので、google_analytics が有効になる。


## 投稿日の追加

ブログは、いつ書かれたかが重要だが、デフォルトでは表示されない。

テンプレート中で、`page.date`という変数で、記事の日付(ファイル名から得られる、または YAML front matter に書かれる)を取得することができる。
これを `date: "%Y-%m-d"`のようにフォーマットしてやれば良い。フォーマット文字列は、strfmt に準じている。

以下を `_layout/post.html`に記載する。

```html
  <div class="date">
    Written on \{\{ page.date | date: "%Y-%m-%d" \}\}
  </div>
```

※ テンプレートの記法(Liquid)がうまく書けないのでエスケープしている。

## tag

ブロクにはいろいろな種類の記事があるが、関連する他の記事を読みたくなることがある。そういう時は、記事の tag または category が助けになる。
Jekyll の YAML front matter には tags: でタグが書けるが、標準のテンプレートでは活用していない。

各記事の YAML front matter にタグ情報を書く。
`tags:`で始まって、` `区切りで複数書ける。

```
---
layout: post
title: GitHub で Blog を作る
category: blog
tags: github jekyll
---
```

テンプレート中で `page.tags`の各要素に対して tag を生成する。tag のリンク先は `/tags.html` の中とする。

```html
  <div class="tag">
    \{\% for tag in page.tags \%\}
      <a href="\{\{ site.baseurl \}\}/tags#\{\{ tag \}\}">\{\{ tag \}\} </a>
    \{\% endfor \%\}
  </div>
```

`/tags.html` では、`site.tags`からタグ一覧を拾って、そのタグを持つページを `tag[1]`で逆リンクする。
```html
\{\% for tag in site.tags \%\}
<article>
  <h1 id="\{\{ tag[0] \}\}">\{\{ tag[0] \}\}</h1>
  <ul>
    \{\% for post in tag[1] \%\}
    <li><a href="\{\{ post.url \}\}">\{\{ post.title \}\}</a></li>
    \{\% endfor \%\}
  </ul>
</article>
\{\% endfor \%\}

```

[http://qiita.com/mnishiguchi/items/fa1e8fd2e893ea801ce8](http://qiita.com/mnishiguchi/items/fa1e8fd2e893ea801ce8)、
[http://genjiapp.com/blog/2013/11/21/simple-tags-page-for-jekyll.html](http://genjiapp.com/blog/2013/11/21/simple-tags-page-for-jekyll.html)
を参照した。

