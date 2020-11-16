---
title: "Docker/Kubernetes 実践コンテナ開発入門@32日目"
date: 2020-11-16T21:33:44+09:00
draft: false
---

[Docker/Kubernetes 実践コンテナ開発入門：書籍案内｜技術評論社](https://gihyo.jp/book/2018/978-4-297-10033-9)

## 6.7 オンプレミス環境でのKubernetesクラスタの構築
* k8sはOSSのため、オンプレミス環境においてもクラスタを構築可
* kubesprayを使用してクラスタ構築

## 6.8 kubesprayでKubernetesクラスタを構築する
* テキストに書いてあるサーバを用意するのが難しいのでここからはメモだけ。

###	6.8.1 クラスタとして構築するサーバの準備
* master
  * nodeの管理、Serviece、Podといったリソースの管理を行う司令塔
  * クラスタの可用性を確保するために3台配置することが推奨
  * オンプレミス環境では開発者が構築・メンテナンスを行う必要がある。
* node
  * Podのデプロイ先
* ops
  * kubesprayを実行するための作業用サーバ
  * k8sのクラスタには入らない
  * opsからAnsibleを利用してkubesprayを実行 → MasterとNodeをk8sクラスタとして構成

###	6.8.2 opsのSSH公開鍵の登録
* opsにて公開鍵(.pub)と秘密鍵を作成する。
* 公開鍵はログインするmasterとnodeサーバに置く

###	6.8.3 IPv4フォワーディングを有効にする
* /etc/sysctl.confのnet.ipv4.ip_forwardのコメントアウトを外す
* 変更反映のため、サーバ再起動

* ops
  * Ansibleとnetaddrをインストール
  * Ansibleのセットアップ

###	6.8.4 クラスタの設定

* inventory/mycluster/hosts.ini
  * クラスタを構成するサーバの設定
  * MasterとNodeを設定できる
* inventory/mycluster/group_vars/k8s-cluster.yml
  * kubernetesのバージョンや、Dashboardのプリインストールの設定等を行うことができる

###	6.8.5 クラスタ構築の実行
* ansible-playbookコマンドを実行すると、クラスタの構築が開始される
* およそ20分～30分で構築が完了する
* 完了したら、kubectlコマンドを実行できるか確認する

## 今日の学び
* 今日はテキストをなぞるだけであったがローカルでもk8sのクラスタ構築ができることを知れた。
* ローカルでもできるとはいえ、メンテナンス等を考えるとクラウドのほうがよさそうなイメージではある。
