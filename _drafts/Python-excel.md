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

## Windowsへのインストール

調べてみたところ、Python はバージョン依存がめんどくさそうだったので、anaconda でまとめてインストールした。バージョン 2系列と3系列化があるが、今から始めるなら3系列で良いだろす。インストーラをダウンロードしてインストールする方法もあるが、chocolaty を使ってみた。chocolaty gui で検索すると `anaconda3` というパッケージがある。GUIからインストールしても良いし、`cinst anaconda3`でもよい。

anaconda をインストールすると、環境変数がセットされたPython コンソールだけでなく、`scipy`などの有名ライブラリ、Jupyter Notebook なども合わせてインストールされる。

とりあえず、Windows上で開発する。`openpyxl`は`pip`でインストールするが、それ以外は標準ライブラリでコトが足りる。

### VS Code

`*.py`ファイルを開くと、拡張機能のインストールを提案されるので、それをそのまま使う。書いている端から文法エラーをチェックしてくれる。いちいち実行しなくてもよいのが便利だ。また、ステップ実行などのデバッグ機能もあるし、タスク機能を使ってユニットテストを容易に実行できるのもよい。

#### タスク機能

メニューの「タスク」からタスクを構成することができる。Pythonの場合ビルドは不要なので、テストタスクを構成する。口述のユニットテストディスカバリーコマンドを関連付けておけばよい。

### `openpyxl`

Excelファイルを扱うライブラリ。`pip3 install openpyxl`で入る。PCにExcelがインストールされていなくても、LinuxでもOK。
文字情報だけでなく、装飾も取り扱い可能。

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

```python
form = cgi.FieldStorage()
reader_from_string = csv.reader(form.getvalue('input', 0).decode('utf-8').strip().spltlines())
with open(filename, 'r') as f:
    reader_from_file = csv.reader(f)
```

### ツリー構成

* ルートディレクトリは index.cgiとか。
* src/ ディレクトリ以下にライブラリを置く。今回は複数のツールを作成するのだが、共通部分はライブラリとしてまとめる。
* test/ 以下にテストコースを置く。

### `if __name__ == '__main__'`

個々のファイルは、モジュールとして import することもできる。スクリプトとして直接実行されたときでは、このif 文が成立するので、そのライブラリの動作確認用の「main」ルーチンを書いておくことができる。簡単なコマンドとして実行テストすることができるライブラリは便利だ。

### OSのパスの取り扱い

WindowsでもLinuxでも動作するとなると、ファイルを指定するときなどでパスの区切り文字が`\`か`/`かなどで、さらに`\`の場合はエスケープをどうするかなど、面倒だ。Pythonには`os.path`モジュールがあり、標準ライブラリでそのへんの面倒を見てくれる。

### ライブラリディレクトリの指定の方法

Pythonではライブラリのディレクトリをパッケージとして扱うためには、そのディレクトリの中に`__init__.py`というファイル(からで良い)を作っておく必要がある。そして、プロジェクトのトップディレクトリを`sys.path`に`append`しておく。そうすれば、トップディレクトリからの相対パスで、システムのモジュールと同様な感じで`import`することができる。

トップディレクトリの`index.cgi`の場合
```python
sys.path.append(os.path.dirname(os.path.abspath(__file__)))
```
`sys/`ディレクトリの中のライブラリファイルの場合

```python
sys.path.append(os.path.join(os.path.dirname(os.path.abspath(__file__)),os.path.pardir))
```

### CGI

自分ひとりが使うツールの場合は、Python スクリプトをコマンドプロンプトから使えば良い。しかし職場の関係者全員に、Python をインストールしてコマンドラインツールを使わせるのは難しい。CGIとして開発サーバで動かす方法、(PyQtなどで)GUIを付けて(cx_Freezeなどで)Windows実行形式にする方法などが考えられる。現時点で仕様が固まっていないので、アップデートが容易な前者の方法を取る。

Windows上でコマンドラインツールとして機能を開発し、そのスクリプトを、CGI側ではライブラリとして呼び出すようにする。Windows上からGitサーバにpushし、Linuxサーバでpullする。ファイルパーミッションをセットするスクリプトを走らせれば、CGIが実行可能だ。


標準ライブラリに `cgi`モジュールがある。

* `form = cgi.FieldStorage()` とすると `form`にパースされたデータが入る。
* `value = form.getfirst('key')` とすると、`value`に`key`で指定したフォームデータが得られる。`value=form['key']`としたいところだが、`key`に対応する値が2つ以上あった場合、リストが帰ってきて、型不整合を起こすので、常に`getfirst`を使うのが良いだろう。デフォルト値も持たせることができる。
* フォーム側を`multipart/form-data`にしておけば、ファイルのアップロードも対応可能だ。
* エラー表示のために`import cgitb`せよと書かれているが、役に立つケースは半分ぐらい。結局`sudo tail /var/log/apache2/error.log`するはめになることが多い。

### Template

Pythonの`string`クラスは標準でテンプレート機能を持っている。`from string import Template`とすれば使えるようになる。

テンプレートファイルの方では、置き換えたいものを`${key}`としておく。プログラムの方では、辞書に置き換えたい値を格納しておき、`safe_substitute()`を呼ぶ。`safe_substitute`は、テンプレート中にあって、辞書に無いキーワードを適切に処理してくれる。

```python
dic['key']='value'
with open(template_file, encoding='utf-8', mode='r') as f:
    template = Template(f.read())
    print(template.safe_substitute(dic))
```


### Unicodeについて

Python3はUnicodeにネイティブに対応している。Windows上では、スクリプト中の文字列、ファイルから読み書きする文字列はUTF-8として扱われる。Linuxのコマンドラインでもそうだ。しかし、CGIとして実行すると状況がことなり、ASCIIバイト列として扱われてしまう。ファイルから読み込んだUTF-8文字列(例えば出力するHTMLのテンプレート)をstdoutに出力しようとすれば「ASCIIを期待しているのにUTF-8が来た」というエラーになる。

原因は、ユーザ環境では通常`LANG=UTF-8`という環境変数がセットされているが、CGI環境ではセットされていないか`LANG=C`なためだ。CGI環境でUTF-8入出力を行うためには、次のような一文が必要だ。

```python
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
```


### テスト

Pythonには標準でユニットテストの機能がある。`test/`以下にテストコードをおいて、`src/`以下のユーザコードをテストするには、`./`を基準として`src/`を読み込むようにモジュール検索パスを指定する。

そのうえで、テスト対象の識別子をすべてimportすればよい。

unittestの書き方は
```python
import unittest

from src.file_under_test import *

class Test_module_under_test(unittest, TestCase):
    def test_function_under_test(self):
        # テスト入力データをセット
        # テスト対象関数の実行
        # 期待値のセット
        # self.assertEqual(expected, ret)
```

テストディスカバリーのやり方は次のとおり。対象ディレクトリ以下のUnitTestを拾い集めて、全て実行してくれる。

```
python -m unittest discover
```


