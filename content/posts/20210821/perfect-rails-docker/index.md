---
title: "パーフェクトRuby on Railsで何度もrails newできる環境をDockerで作る"
date: 2021-08-21T19:46:06+09:00
draft: false
---

## モチベーション
パーフェクトRuby on Railsを1～3章まで進めてみたのですが、とにかくrails newの頻度が多いです。  

ローカルを汚したくないのでDocker(docker-compose)で完結するようにやっていたのですが、  

rails newするために色々手間だったので学習効率をあげるためにもう少し簡単にできないか模索してみました。  

## 参考にした記事

[Dockerを使って手元の環境にRubyやRailsを入れずにrails newした結果を手に入れる \- コード日進月歩](https://shinkufencer.hateblo.jp/entry/2020/08/06/233446)

もともとdocker-compose構成で[こちらのリポジトリ](https://github.com/snyt45/perfect-rails2)でやっていましたがDockerfile一つでもいけるのかと思ったのでDockerfileのみでやるようにしてみました。

## 最終的なDockerfileのリポジトリ

Dockerfileはgithubにあげておきました。

[snyt45/perfect\-rails\-docker: 何度もrails newできるコンテナ設定](https://github.com/snyt45/perfect-rails-docker)

## やりたいこと

- 1つのコンテナの中でrails newを何回もできるようにしたい。
- DBはSQLite3でOK ※MySQLを使ったセットアップすると少し面倒そうなので…
- rails server起動したらlocalhost:3000で接続できること
- ファイル編集はしたいのでホストのDockerfileと同じディレクトリをコンテナにマウントする

## やったこと

### Dockerfile

ほぼ参考にした記事通りですが、rails newをしないようにしています。  

rails6に必要なrubyとnodejsとyarnを入れて、rubyを入れたら使えるgemコマンドでrailsをインストールするところまでのイメージを作るようにしてみました。  

また、appディレクトリを作っていますがこの配下に複数プロジェクトを配置するイメージです。

```
ARG ruby_version
FROM ruby:${ruby_version}

# nodejsをインストール
RUN curl -LO https://nodejs.org/dist/v12.18.3/node-v12.18.3-linux-x64.tar.xz
RUN tar xvf node-v12.18.3-linux-x64.tar.xz
RUN mv node-v12.18.3-linux-x64 node
ENV PATH /node/bin:$PATH

# yarnをセットアップする
# 公式ドキュメントまま https://classic.yarnpkg.com/ja/docs/install/#alternatives-stable
RUN curl -o- -L https://yarnpkg.com/install.sh | bash
ENV PATH /root/.yarn/bin:/root/.config/yarn/global/node_modules/.bin:$PATH

ARG rails_version
RUN gem install rails -v "${rails_version}"

RUN mkdir /app
ENV APP_ROOT /app
WORKDIR $APP_ROOT
```

### イメージ作成

カレントディレクトリのDockerfileを使用。

イメージ名はperfect-railsで、rubyは2.6.3、railsは6.0.1のバージョンを指定しています。

```
docker build . -t perfect-rails --build-arg ruby_version=2.6.3 --build-arg rails_version=6.0.1
```

### イメージからコンテナ起動

perfect-railsというイメージを元にコンテナを起動。

-dでバックグラウンド実行しています。

-tを付けておかないとコンテナが停止してしまうのでつけています。

-pでホスト側の3000番ポートにアクセスがあったら、コンテナの3000番ポートにポートフォワーディングさせるようにしています。

--nameでコンテナの名前をperfect-railsにしています。

-vでカレントディレクトリをコンテナのappディレクトリにマウントします。

```
# fish
docker run -d -t -v (pwd):/app -p 3000:3000 --name="perfect-rails" perfect-rails
```

コンテナが起動していることを確認する。

```
❯ docker ps
CONTAINER ID   IMAGE           COMMAND   CREATED          STATUS          PORTS
  NAMES
b7c53f1277cc   perfect-rails   "irb"     30 seconds ago   Up 29 seconds   0.0.0.0:3000->3000/tcp, :::3000->3000/tcp   perfect-rails
```

### 複数プロジェクト作成、サーバー起動、localhost:3000に接続できることを確認する

#### 1つ目のプロジェクト作ってみる

rails newする。

databaseはデフォルトのsqlite3でいいので何も指定しない。

```
docker exec -it perfect-rails /bin/bash
rails new sample_app_1
```

sample_app_1フォルダが作成されました。
```
.
├── Dockerfile
└── sample_app_1
```

サーバー起動して、localhost:3000に接続できることを確認します。

```
cd sample_app_1
rails s -p 3000 -b '0.0.0.0'
```

接続できました！

#### 2つ目のプロジェクト作ってみる

rails serverは停止しておきます。

さっきと同じ手順で実行します。

rails newして新しいプロジェクトを作成します。

```
rails new sample_app_2
```

新しくsample_app_2フォルダが作成されました。
```
.
├── Dockerfile
├── sample_app_1
└── sample_app_2
```

サーバー起動して、localhost:3000に接続できることを確認します。

```
cd sample_app_2
rails s -p 3000 -b '0.0.0.0'
```

同じく接続できました！

### コンテナを停止して再起動する

運用してはコンテナを停止して再起動して使います。

```
docker stop perfect-rails
docker strat perfect-rails
```

docker start後にdocker psしてみると、ポートフォワーディングも設定された状態のコンテナが起動していることが確認できます。

```
❯ docker ps
CONTAINER ID   IMAGE           COMMAND   CREATED          STATUS         PORTS
 NAMES
b7c53f1277cc   perfect-rails   "irb"     16 minutes ago   Up 3 seconds   0.0.0.0:3000->3000/tcp, :::3000->3000/tcp   perfect-rails
```

### まとめ

パーフェクトRuby on Railsでrails newが出てきてもこれで1つのコンテナ内で対応できるようになりました！

これでいくらでも試せる環境ができたので学習効率があがりそう。
