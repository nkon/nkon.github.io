---
layout: post
title: Python で Excel を操作する
category: blog
tags: windows python excel
---

Python初心者がアプリを作ったときに引っかかったことをメモしておく。

# 背景

『めんどうなことはPythonにやらせよう』という書籍がある。非プログラマ向けに、書籍前半では、WindowsへのPythonのインストールの方法、基本的な文法とデータ構造を述べている。後半ではいろいろな拡張ライブラリを使ってWindowsのアプリを使って行う操作を、Pythonにやらせている。

そこで紹介されていたライブラリに`openpyxl`というものがある。PythonからExcelを扱うライブラリだ。ちょうど同時期に、職場で部品表をどうるすか、という課題があった。CADが部品表を生成するが、それは電子的なものであり、監査向けの部品表書類を作らなければならないのだ。ちょうど覚えたてのPythonを使ってツールを作ってみた。

## インストール

調べてみたところ、Python はバージョン依存がめんどくさそうだったので、anaconda でまとめてインストールした。インストーラをダウンロードしてインストールする方法もあるが、chocolaty を使ってみた。chocolaty gui で検索すると `anaconda3` というパッケージがある。GUIからインストールしても良いし、`cinst anaconda3`でもよい。

anaconda をインストールすると、環境変数がセットされたPython コンソールだけでなく、`scipy`などの有名ライブラリ、Jupyter Notebook なども合わせてインストールされる。

## VS Code

とりあえず、Windows上で開発する。`openpyxl`は`pip`でインストールするが、それ以外は標準ライブラリでコトが足りる。

### `openpyxl`

Excelファイルを扱うライブラリ。`pip3 install openpyxl`で入る。PCにエクセルがインストールされていなくても、LinuxでもOK。

* Workbookの新規作成:`wb=openpyxl.Workbook()`
* Workbookの読み込み:`wb=openpyxl.load_workbook(filename)`
* Workbookのストリームからの読み出し:
* Workbookの保存:`wb.save('filename')`
* Worksheetの指定:`ws=wb.active`
* セルの読み込み:`a=ws['A1].value`
* セルの書き込み:`ws['B2'].value='Hello World'`
* セルの書き込み2:`ws.cell(row=1,column=3).value='Hello World'`


### `csv`

CSVファイルを扱うライブラリ。Excelのデータをテキストでやり取りするときには、いまだにCSVがよく使われる。CSVは、コンマやダブルクォーテーションのような可視文字を区切りに使うので、パースがややこしい。個人的にはTSV(タブ区切り)のほうがいいと思うのだが。

CSVは標準でライブラリがあるのでありがたく使わせていただく。
`csv`ライブらいでは、CSVファイルをオープンして、`reader`というストリームに割り付けて読み込んでいく。ストリームであれば、ファイルではなく文字列でも良いので、Web Interfaceにしたときにも扱いやすい。

### ツリー構成

* ルートディレクトリは index.cgiとか。
* src/ ディレクトリ以下にライブラリを置く。今回は複数のツールを作成するのだが、共通部分はライブラリとしてまとめる。

### `if __name__ == '__main__'`

個々のファイルは、ライブラリとして import することもできる。スクリプトとして直接実行されたときでは、このif 文が成立するので、そのライブラリの動作確認用の「main」ルーチンを書いておくことができる。簡単なコマンドとして実行テストすることができるライブラリは便利だ。

### ライブラリディレクトリの指定の方法

### OSのパスの取り扱い

### CGIライブラリ

### Unicodeについて

