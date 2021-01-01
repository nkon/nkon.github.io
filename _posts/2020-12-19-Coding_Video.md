---
layout: post
title: コーディング動画(Rustで関数電卓を作る)
category: blog
tags: rust coding 動画 youtube ffmpeg
---


この冬もコロナで篭もることになりそうだ。関数電卓作成のプロジェクトを行っている。プロジェクトの説明、設計や実装については別途記事にしたいと考えているが、現時点では執筆中だ。一方、最近、他人のコーディング動画を見る機会があった。私の職場にはペアプログラミングの習慣はないので、他人のコーディングを見る機会は少なかったが、動画からは、記事だけでは伝わらない・ナルホドと関心する点が多かった。自分もコーディング動画を配信すれば、コーディングの進め方やツールの使い方などで参考にしていただけることもあるだろう。

<iframe width="560" height="315" src="https://www.youtube.com/embed/9JqJR-TvJzg" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>

埋め込み再生ではなくYouTubeのサイトに行ってHD画質で再生すると文字が潰れない。

プロジェクトのリポジトリはGitHubに公開しているので、そこからソースコードや履歴を参照することができる。
動画の対象としているのは、ちょうど次のコミットだ。

[https://github.com/nkon/rc-rs/commit/46fae67ce391f72a9646022de06ec227bc1aab27](https://github.com/nkon/rc-rs/commit/46fae67ce391f72a9646022de06ec227bc1aab27)

じつは、その後すぐにリファクタリングで実装を簡素化している。

[https://github.com/nkon/rc-rs/commit/e52f889d4a0025c753e98d36362866ea6625c836](https://github.com/nkon/rc-rs/commit/e52f889d4a0025c753e98d36362866ea6625c836)


ここでは、動画のレコーディングや編集について説明する。


## 録画

ゲーム実況用の[OBS Studio](https://obsproject.com/ja)なども有名だが、今回は[SimpleScreenRecorder](https://www.maartenbaert.be/simplescreenrecorder/)というツールを使った。画面を録画するとともに、マイクでの入力も同時に録音できる。起動するとWizerd形式の設定画面が立ち上がり、簡単に画面録画が可能だ。

## 音声の編集

そのままYouTubeにアップしたら音声レベルが小さすぎた。

まずはffmpegで音声ファイルを取り出す。

```
$ ffmpeg -i live-coding-2020-12-19_12.59.52.mkv -vn -acodec copy live-coding-2020-12-19_12.59.52.mp3
```

[Auphonic](https://auphonic.com/landing)というサービスで音声レベルを調整する。PodCastなどで使われるサービスだが、AI解析で音量レベルを調整したりノイズ除去をしたりが可能だ。月2hの無料枠がある。足りない場合は時間を追加購入できる。

ffmpegで音声と映像をマージする。

```
$ ffmpeg -i live-coding-2020-12-19_12.59.52.mkv -i Downloads/live-coding-2020-12-19_12.59.52.mp3 -c:v copy -c:a copy -map 0:v:0 -map 1:a:0 output.mp4
```


[Kdenlive](https://kdenlive.org/en/)などで字幕を付けることもできる。今回はできるだけ動画中で喋って解説したので字幕は付けない。


これをYouTubeにアップすればOK。

## 動画の内容を理解するための補足説明

作っているのはコマンドライン型の関数電卓。整数、浮動小数点数、複素数が扱える。電卓の構造としては、基本通り、次の流れになる。

(1)トークナイザで入力文字列をトークン列に変換する。

(2)手書きの再帰降順パーサでAST(`Node`という構造体で実装される)を構築する。

(3)ASTを評価して計算結果を得る。

今回の動画の範囲は「`sqrt()`関数の追加」なので、(3)の評価機に関する機能追加だ。

`sqrt()`のような言語組込みの関数は、次のように実装される。

(1) `sqrt`という識別子を内蔵の辞書に登録しておく。

(2) `sqrt`という識別子がやってきたときに、`sqrt`の計算を実行する。

(3) 今回は`sqrt`の計算(おそらくlibmなどにあるだろう)を実行するのではなく、すでに実装されているべき乗演算子(`^`)を使った書き換えにより、`sqrt`関数を実装する。つまり、`sqrt(a)`という関数を、`a^0.5`と書き換える。

