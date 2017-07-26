---
layout: post
title: nvdia-docker で ディープラーニング用の環境を作る
category: blog
tags: docker, ai, ubuntu
---

AIでいろいろ遊んで見るために、まずは環境構築する。

変化が早くいろいろな情報があっという間に古くなる中で、執筆時点(2017年7月)で、最も使いやすい(作りやすい・維持しやすい・情報を得やすい)環境を目指す。遊ぶ前に環境構築で疲れてしまうわけにはいかない。


前提として、手元の環境は次の通り。

* CPU: Core i7-2600(Sandy Bridge)
* Main Memory: 32GiB
* OS: Ubuntu 16.04
* GPU: GeForce GTX 1050
* nvidia driver: 375.66

ここに環境を構築するのにいくつかの手段がある。

## CPU のみ
すこしググッてみたところ、簡単な解説は VirtualBox か RasPi 上に Ubuntu + pip + python3 の環境を作っていることが多い。環境構築としては簡単だが、全く遅くて実用的ではないのと、1050とはいえせっかくあるGPUを使わないのはもったいない。

## GPUを使う
Deep Lerning で GPU を使う時点で、業界標準としては Ubuntu + nvidia となる。しかし、Cuda ドライバのインストールが、まだこなれていなさそうだ。nvidia driver への依存性、Xのドライバとの干渉など。Deep Lerning 専用機ではなく、デスクトップ環境兼用機なので、極力トラブルは起こしたくない。

よって Docker 上に環境を構築する。幸い、最近、nvidia-docker が登場し、これからは nvdia-docker が標準的になりそうだ。

## nvidia driver のセットアッブ

X が GPU で動いている場合、nvidia driver はすでに正常に動作しているはずだ。
次のようになれば問題ない。

```
$ nvidia-smi
nkon@crimson:~/src/docker/example/test$ nvidia-smi
Sun Jul 16 22:23:24 2017
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 375.66                 Driver Version: 375.66                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 1050    Off  | 0000:01:00.0      On |                  N/A |
| 46%   35C    P0    35W /  75W |    351MiB /  1997MiB |      1%      Default |
+-------------------------------+----------------------+----------------------+
                                                                               
+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID  Type  Process name                               Usage      |
|=============================================================================|
|    0      1453    G   /usr/lib/xorg/Xorg                             200MiB |
|    0      4190    G   cinnamon                                       146MiB |
|    0      4909    G   /usr/lib/firefox/firefox                         1MiB |
+-----------------------------------------------------------------------------+
```

## docker のセットアッブ

通常の docker のセットアッブをしておく。
`apt-get install docker.io`で良かったと思う。

## nvidia-docker のセットアッブ
本家の解説([https://github.com/NVIDIA/nvidia-docker/wiki/Installation](https://github.com/NVIDIA/nvidia-docker/wiki/Installation))の通りにやれば、問題なく終了するはずだ。

### 動作確認

まずは、本家のサンブル([https://github.com/NVIDIA/nvidia-docker/blob/master/samples/ubuntu-16.04/nvidia-smi/Dockerfile](https://github.com/NVIDIA/nvidia-docker/blob/master/samples/ubuntu-16.04/nvidia-smi/Dockerfile))を取ってきて実行する。


```
$ vim Dockerfile
〜上の内容を貼り付ける〜

FROM nvidia/cuda:8.0-devel-ubuntu16.04

CMD nvidia-smi -q

ZZ
```

docker image を作る。
```
$ sudo docker build -t nvidia/cuda .
```

実行すると `nvidia-smi -q`を実行して終了するはずだ。

```
$ sudo nvidia-docker run --rm -i -t nvidia/cuda

==============NVSMI LOG==============

Timestamp                           : Sun Jul 16 13:58:42 2017
Driver Version                      : 375.66

Attached GPUs                       : 1
GPU 0000:01:00.0
    Product Name                    : GeForce GTX 1050
    Product Brand                   : GeForce
    Display Mode                    : Enabled
    Display Active                  : Enabled
    Persistence Mode                : Disabled
    Accounting Mode                 : Disabled
    Accounting Mode Buffer Size     : 1920
    Driver Model
〜続く〜
```

## TensorFlow + Keras の環境を追加する

[http://archive.is/RcstB](http://archive.is/RcstB) を参照した。

Dockerfile を次のように編集する。

* cuDNN 付きのイメージをベースとする。参照先に書かれているように、メンバー登録の手間を省くため、cuDNN5 付きのイメージを利用する。
* pip で python3 を入れる。
* GPU付きの TensorFlow と Keras を入れる。

```
FROM nvidia/cuda:8.0-cudnn5-runtime

RUN apt update && apt install -y python3-pip
RUN pip3 install tensorflow-gpu keras
```

Docker image を作る。

```
$ sudo docker build -t nvidia/cuda .
```

コンテナを起動する。
```
$ sudo nvidia-docker run -it nvidia/cuda
```
この状態で、シェルが起動しているので、コマンドを実行する(Save しないと忘れる)。

```
# apt install wget
# wget https://raw.githubusercontent.com/fchollet/keras/master/examples/mnist_cnn.py
# python3 mnist_cnn.py
```

CUDAのコンパイルまでは CPU を使っているが、それが終わったらCPUの負荷が減り、すごいスピードで学習が進み、ファンの音がCPUの音からGPUの音に変わればうまくいっている。

## chainer の環境を追加する

書籍となっている解説書では chainer を使ったものも多い。
公式([https://github.com/chainer/chainer/blob/master/docker/python3/Dockerfile](https://github.com/chainer/chainer/blob/master/docker/python3/Dockerfile))ではうまく動かない。いろいろ試したが次で動いた。これは執筆時点での動作実績であり、安心して使える気がしないのは、やはりバージョンに敏感すぎるだろう。

```
FROM nvidia/cuda:8.0-cudnn5-devel

RUN apt-get update -y && \
    apt-get install -y --no-install-recommends \
    python3-dev \
    python3-pip && \
    rm -rf /var/lib/apt/lists/* /var/cache/apt/archives/*

RUN pip3 install --upgrade pip
RUN pip3 install setuptools
RUN pip3 install cupy chainer
```

同じく、MNISTでテストする。`--gpu=0` で0番目のGPUを使うことを指示している。

```
$ sudo nvidia-docker run -it nvidia/cuda
(コンテナのシェル)
# apt-get update
# apt-get install wget python3-matplotlib
# wget https://raw.githubusercontent.com/chainer/chainer/master/examples/mnist/train_mnist.py
# python3 train_mnist.py  --gpu=0
```

自分で、コードを書いて遊ぶために、ホストのディレクトリをマウントして起動する。

今更だが、`-i`はインタラクティブシェルを起動、`-t`は端末を接続だ。

```
$ sudo nvdia-docker run -v [ホストディレクトリの絶対パス]:[コンテナの絶対パス] -it nvidia/cuda
```
