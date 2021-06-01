---
title: "docker-compose up -dではなくdocker-compose upを使おう"
date: 2021-06-01T22:32:56+09:00
draft: false
---

自分のエラーではないが、同僚がdocker-compose up時に下記エラーが出て解決まで至ったので忘れないうちにメモ。

## エラーログ
このエラーが出る前に、`docker-compose down --volumes`を行いエイリアスコマンドで`docker-compose up -d`を実行していた。下記は`docker-compose up -d`時のエラー。

```
ERROR: for www_spring_1  Cannot start service spring: OCI runtime create failed: container_linux.go:367
Creating www_rails_1  ... done
Creating www_https-portal_1 ... done
ERROR: for spring  Cannot start service spring: OCI runtime create failed: container_linux.go:367: starting container process caused: exec: "spring": executable file not found in $PATH: unknown
ERROR: Encountered errors while bringing up the project.
```

## エラーログ解消のためにやったこと

`docker-compose down --volumes`は、コンテナの停止&破棄とdocker-compose.ymlファイルのvolumesセクションの名前付きvolumeを削除する。

例えば、下記の場合は`bundle`と`node_modules`のvolumeが削除される。

```
version: "3"
services:
  db:
    ...
  rails:
    ...
  spring:
    ...
  redis:
    ...
volumes:
  bundle:
  node_modules:
  ...
```

次に、`docker-compose up -d`をエイリアスコマンドで実行していた。
ここが今回の原因発覚を遅らせたコマンド。

`-d`はバックグラウンドで実行されるため、実行ログがコンソール上には出ず実行結果としてdone または error のような形で結果がわかる。

```
Creating redis_1 ... done
Creating db_1    ... done
Creating rails_1  ... done
Creating spring_1 ... error
```

このとき、名前付きボリュームを削除したためrailsコンテナ起動時にbundle installが改めて走っていたが、上記のようにdoneと表示されていたため問題ないものと思いすぐに`docker-compose down --volumes`を行っていろいろと解決のため試していた。

そのため、bundle installがいつまで経っても終わらず、springコンテナはこのvolumeを使うように指定していたため起動時には必要なvolumeがないためエラーを吐いていた。

`docker-compose up`でログを確認しながら、bundle install等が終わったことを確認しvolumesができたことを確認し再度`docker-compose up`を行ったらspringコンテナを起動することができた。

バックグラウンドで起動すると裏で何が行われているか確認ができず、原因発覚が遅れるのでこういうときはフォアグランドで実行しようという話でした(特にエイリアスコマンドなどで普段コンテナ起動していると忘れがち)。
