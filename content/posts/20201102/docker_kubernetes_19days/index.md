---
title: "Docker/Kubernetes 実践コンテナ開発入門@19日目"
date: 2020-11-02T00:30:50+09:00
draft: false
---

[Docker/Kubernetes 実践コンテナ開発入門：書籍案内｜技術評論社](https://gihyo.jp/book/2018/978-4-297-10033-9)

今日はNginxの構築をできるところまでやっていきます。

## 4.4 Nginxの構築
* クライアントから受け取ったHTTPリクエストを、リバースプロキシを用いてバックエンドWebに転送
* WebからAPI側へのリクエストを転送するためにも使用

* リバースプロキシって？
  * クライアントとサーバの中継
  * クライアントに対してはサーバとして振る舞い、サーバに対してはクライアントとして振る舞う
  * 中継にリバースプロキシをはさむことで、負荷分散・キャッシュや圧縮による高速化など様々なメリットがあるよう。
  * [リバースプロキシ（Reverse Proxy）：90秒の動画で学ぶITキーワード \- ＠IT](https://www.atmarkit.co.jp/ait/articles/1608/25/news034.html#:~:text=%E3%83%AA%E3%83%90%E3%83%BC%E3%82%B9%E3%83%97%E3%83%AD%E3%82%AD%E3%82%B7%EF%BC%88Reverse%20Proxy%EF%BC%89%E3%81%A8,%E3%81%AB%E3%82%88%E3%81%8F%E5%88%A9%E7%94%A8%E3%81%95%E3%82%8C%E3%82%8B%E3%80%82)

* git cloneする
```
$ git clone https://github.com/gihyodocker/todonginx
```

* trreでディレクトリ構造確認
```
$ tree .\todonginx\ /f
フォルダー パスの一覧:  ボリューム Windows
ボリューム シリアル番号は 6607-7B7E です
C:\USERS\SNYT45\DESKTOP\DOCKER_KUBERNETES 実践コンテナ開発入門\4-2\TODONGINX
│  Dockerfile
│
└─etc
    └─nginx
        │  nginx.conf.tmpl
        │
        └─conf.d
                log.conf
                public.conf.tmpl
                upstream.conf.tmpl
```

* フロントエンドとAPIのそれぞれの前に置くServicesが存在する。同じDockerfileから作成。
* 今回は、APIの前におくNginxのServiceを作成。

### 4.4.1 nginx.confを構築する
* ベースイメージとしてnginx:1.13を使用。

* /etc/nginx/nginx.conf
  * Nginxの設定の起点となるファイル

* Nginxの性能をチューニングする上で注目すべき項目
  * worker_processes
    * Nginxのworkerプロセス数
  * worker_connections
    * workerプロセスが開ける最大コネクション数
  * keepalive_timeout
    * クライアントからの接続を維持する秒数
  * gzip
    * レスポンスのGZIP圧縮を有効にするか

* 性能に関わる値は固定値を定義ではなく、環境変数化したい
```
user nginx;
worker_processes 1; # ← ①

error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
  worker_connections 1024; # ← ②
}
http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';

  access_log /var/log/nginx/access.log main;

  sendfile on;
  #tcp_nopush on;

  keepalive_timeout 65; # ← ③

  #gzip on; # ← ④

  include /etc/nginx/conf.d/*.conf;
}
```

* 環境変数で制御できるようにentrykitのテンプレート機能(tmplファイル)を使う
* entrykitを使うと、コンテナ実行のタイミングでテンプレートと環境変数に設定された値からファイルを生成(render)できる

* 次のように環境変数を埋め込む
```
{{ var "環境変数名" }}
{{ var "環境変数名" | default "デフォルト値" }}
```
* Vue.jsの”Mustache” 構文(二重中括弧)に似ている！

* etc/nginx/nginx.conf.tmpl
  * nginx.confを生成するためのテンプレートファイル
  * ⑤のincludeによって/etc/nginx/conf.d/*.confに合致するファイルをロードできる → 設定ファイルの追加は/etc/nginx/conf.dにファイルを追加していく
```
user  nginx;
worker_processes  {{ var "WORKER_PROCESSES" | default "1" }}; # ← ①

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  {{ var "WORKER_CONNECTIONS" | default "1024" }}; # ← ②
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  {{ var "KEEPALIVE_TIMEOUT" | default "65" }}; # ← ③

    gzip  {{ var "GZIP" | default "on" }}; # ← ④

    include /etc/nginx/conf.d/*.conf; # ← ⑤
}
```

* ログ - etc/nginx/conf.d/log.conf
  * ログフォーマットの定義

* バックエンドサーバの振り分け - etc/nginx/conf.d/upstream.conf.tmpl
  * プロキシ先はNginxのproxy_passディレクティブに直接記述することも可能
  * upstreamディレクトリを定義するメリット
    * プロキシ先のサーバを複数指定してロードバランス
    * 障害時にSorryサーバへプロキシできる設定をできる

* ルーティング - etc/nginx/conf.d/public.conf.tmpl
  * HTTPリクエストのルーティングの設定
  * SERVER_PORT
    * NginxをListenするポート。デフォルト値として80を設定
    * locationディレクティブはバックエンドへのプロキシの設定
      * パスとして/を定義。全てバックエンドにプロキシする
      * proxy_passはupstreamディレクティブで定義したので、http://backendとする
      * LOG_STDOUTの環境変数を利用し、値があれば標準出力に出力し、値がなければファイルに出力

### コラム 環境変数を積極的に使う
* Dockerで挙動を変えたい部分は環境変数化し、デフォルト値を設定する癖をつけておこう

### 4.4.2 NginxのDockerle
```
FROM nginx:1.13

RUN apt-get update
RUN apt-get install -y wget
RUN wget https://github.com/progrium/entrykit/releases/download/v0.4.0/entrykit_0.4.0_linux_x86_64.tgz
RUN tar -xvzf entrykit_0.4.0_linux_x86_64.tgz
RUN rm entrykit_0.4.0_linux_x86_64.tgz
RUN mv entrykit /usr/local/bin/
RUN entrykit --symlink

# ① conf.dのデフォルトファイルを削除し、各種設定ファイルをコピー
RUN rm /etc/nginx/conf.d/*
COPY etc/nginx/nginx.conf.tmpl /etc/nginx/
COPY etc/nginx/conf.d/ /etc/nginx/conf.d/

# ② entrykitでテンプレートから設定ファイルを生成
ENTRYPOINT [ \
  "render", \
      "/etc/nginx/nginx.conf", \
      "--", \
  "render", \
      "/etc/nginx/conf.d/upstream.conf", \
      "--", \
  "render", \
      "/etc/nginx/conf.d/public.conf", \
      "--" \
]

CMD ["nginx", "-g", "daemon off;"]
```

* entrykitの準備後は、①でコンテナの中が次のようなディレクトリ構造になる

```
└── etc
  └── nginx
    ├── conf.d
    │   ├── log.conf
    │   ├── public.conf.tmpl (render後はpublic.conf)
    │   └── upstream.conf.tmpl (render後はupstream.conf)
    └── nginx.conf.tmpl (render後はnginx.conf)
```

* ②のENTRYPOINTでは、entrykitでrenderを実行したいファイルを列挙
* renderで指定するパスは.tmplを付けずに完成後のパスとなるように指定

* ch04/nginx:latestという名前でイメージを作成
```
$ docker image build -t ch04/nginx:latest .
[+] Building 29.0s (16/16) FINISHED
 => [internal] load build definition from Dockerfile                                                                                                                   0.1s 
 => => transferring dockerfile: 713B                                                                                                                                   0.0s 
 => [internal] load .dockerignore                                                                                                                                      0.0s 
 => => transferring context: 2B                                                                                                                                        0.0s 
 => [internal] load metadata for docker.io/library/nginx:1.13                                                                                                          3.3s 
 => [1/11] FROM docker.io/library/nginx:1.13@sha256:b1d09e9718890e6ebbbd2bc319ef1611559e30ce1b6f56b2e3b479d9da51dc35                                                   8.8s 
 => => resolve docker.io/library/nginx:1.13@sha256:b1d09e9718890e6ebbbd2bc319ef1611559e30ce1b6f56b2e3b479d9da51dc35                                                    0.0s 
 => => sha256:b1d09e9718890e6ebbbd2bc319ef1611559e30ce1b6f56b2e3b479d9da51dc35 2.03kB / 2.03kB                                                                         0.0s 
 => => sha256:e4f0474a75c510f40b37b6b7dc2516241ffa8bde5a442bde3d372c9519c84d90 948B / 948B                                                                             0.0s 
 => => sha256:ae513a47849c895a155ddfb868d6ba247f60240ec8495482eca74c4a2c13a881 6.03kB / 6.03kB                                                                         0.0s 
 => => sha256:f2aa67a397c49232112953088506d02074a1fe577f65dc2052f158a3e5da52e8 22.50MB / 22.50MB                                                                       7.2s 
 => => sha256:3c091c23e29d0ddfc902b0be63b1a08a853ef39973f92fab39ad1727eac012bf 22.11MB / 22.11MB                                                                       7.2s 
 => => sha256:4a99993b863683bef1c776732e14d2372f6ed52b48e94783f4a1b58af289db07 201B / 201B                                                                             0.8s 
 => => extracting sha256:f2aa67a397c49232112953088506d02074a1fe577f65dc2052f158a3e5da52e8                                                                              0.7s 
 => => extracting sha256:3c091c23e29d0ddfc902b0be63b1a08a853ef39973f92fab39ad1727eac012bf                                                                              0.3s 
 => => extracting sha256:4a99993b863683bef1c776732e14d2372f6ed52b48e94783f4a1b58af289db07                                                                              0.0s 
 => [internal] load build context                                                                                                                                      0.0s 
 => => transferring context: 2.84kB                                                                                                                                    0.0s 
 => [2/11] RUN apt-get update                                                                                                                                          5.0s 
 => [3/11] RUN apt-get install -y wget                                                                                                                                 5.3s 
 => [4/11] RUN wget https://github.com/progrium/entrykit/releases/download/v0.4.0/entrykit_0.4.0_linux_x86_64.tgz                                                      4.0s 
 => [5/11] RUN tar -xvzf entrykit_0.4.0_linux_x86_64.tgz                                                                                                               0.4s 
 => [6/11] RUN rm entrykit_0.4.0_linux_x86_64.tgz                                                                                                                      0.4s 
 => [7/11] RUN mv entrykit /usr/local/bin/                                                                                                                             0.4s 
 => [8/11] RUN entrykit --symlink                                                                                                                                      0.5s 
 => [9/11] RUN rm /etc/nginx/conf.d/*                                                                                                                                  0.4s 
 => [10/11] COPY etc/nginx/nginx.conf.tmpl /etc/nginx/                                                                                                                 0.0s 
 => [11/11] COPY etc/nginx/conf.d/ /etc/nginx/conf.d/                                                                                                                  0.0s 
 => exporting to image                                                                                                                                                 0.3s 
 => => exporting layers                                                                                                                                                0.3s 
 => => writing image sha256:e750c834a1dc3a8126f460d5f23cc89846eb2da07dfd862ca1407d33c2e703fb                                                                           0.0s 
 => => naming to docker.io/ch04/nginx:latest
```

* 先ほど作成したイメージをlocalhost:5000/ch04/nginx:latestというイメージとして作成
```
$ docker image tag ch04/nginx:latest localhost:5000/ch04/nginx:latest
```

* registryにpush
```
$ docker image push localhost:5000/ch04/nginx:latest
The push refers to repository [localhost:5000/ch04/nginx]
Get http://localhost:5000/v2/: dial tcp [::1]:5000: connect: connection refused
```

### 4.4.3 Nginxを通してアクセスできるようにする

* todo-app.yml
  * 前章で構築したtodo_app_api Serviceの前段にNginxを配置
  * Nginx側でentrykitを使ってコンテナ実行時の環境変数を用いて設定ファイルを生成するため、todo-app.ymlで環境変数を設定するだけで済む

```
version: "3"
services:
  nginx:
    image: registry:5000/ch04/nginx:latest
    deploy:
      replicas: 2
      placement:
        constraints: [node.role != manager]
    depends_on:
      - api
    environment:
      WORKER_PROCESSES: 2
      WORKER_CONNECTIONS: 1024
      KEEPALIVE_TIMEOUT: 65
      GZIP: "on"
      BACKEND_HOST: todo_app_api:8080
      BACKEND_MAX_FAILS: 3
      BACKEND_FAIL_TIMEOUT: 10s
      SERVER_PORT: 80
      SERVER_NAME: todo_app_nginx
      LOG_STDOUT: "true"
    networks:
      - todoapp

  api:
    image: registry:5000/ch04/todoapi:latest
    deploy:
      replicas: 2
      placement:
        constraints: [node.role != manager]
    environment:
      TODO_BIND: ":8080"
      TODO_MASTER_URL: "gihyo:gihyo@tcp(todo_mysql_master:3306)/tododb?parseTime=true"
      TODO_SLAVE_URL: "gihyo:gihyo@tcp(todo_mysql_slave:3306)/tododb?parseTime=true"
    networks:
      - todoapp

networks:
  todoapp:
    external: true
```

* todo_appのStackを更新
```
$ docker container exec -it manager docker stack deploy -c /stack/todo-app.yml todo_app
```

と思いきや、PCを再起動したため最初からやり直す必要がありそう。。

* registry、manager、workerコンテナ起動
```
$ docker-compose up
```

* managerコンテナからサービス一覧確認。
  * docker swarm initからやり直す必要がありそうだ。。。
```
$ docker container exec -it manager docker service ls
Error response from daemon: This node is not a swarm manager. Use "docker swarm init" or "docker swarm join" to connect this node to swarm and try again.
```

* managerコンテナをSwarmのmanagerに設定
```
$ docker container exec -it manager docker swarm init
```

* worker01~03をSwarmクラスタに追加
```
$ docker container exec -it worker01 docker swarm join --token SWMTKN-1-1bt724fy34a1ghxypkxny8fhtz8jsp09z39nswag60bonxejyu-3nvmveic1vyx3vipzwmqesap3 172.18.0.3:2377
This node joined a swarm as a worker.

$ docker container exec -it worker02 docker swarm join --token SWMTKN-1-1bt724fy34a1ghxypkxny8fhtz8jsp09z39nswag60bonxejyu-3nvmveic1vyx3vipzwmqesap3 172.18.0.3:2377
This node joined a swarm as a worker.

$ docker container exec -it worker03 docker swarm join --token SWMTKN-1-1bt724fy34a1ghxypkxny8fhtz8jsp09z39nswag60bonxejyu-3nvmveic1vyx3vipzwmqesap3 172.18.0.3:2377
This node joined a swarm as a worker.
```

* ノードに追加されているか確認
```
$ docker container exec -it manager docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
mc2re0d06joon80xef0qb2s4n     3a2de3f08dbe        Ready               Active                                  18.05.0-ce
bhhwcxzfqil5bmhjt53rh6ona     3cb697256825        Ready               Active                                  18.05.0-ce
w1nrjoj1qs03221ujay7c1a49     9372f731e1c1        Ready               Active                                  18.05.0-ce
1hb7chtd8iywklko5nt5m2ki6 *   e6eb10555c5c        Ready               Active              Leader              18.05.0-ce
```

* サービス確認。まだ何もない。
```
$ docker container exec -it manager docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
```

* ネットワーク確認。todoappネットワークがないことを確認。
```
$ docker container exec -it manager docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
6572733214ed        bridge              bridge              local
52a0ad425489        docker_gwbridge     bridge              local
25faa8a1e997        host                host                local
fjhzcrl5x3l1        ingress             overlay             swarm
2d7c7a1e8113        none                null                local
```

* todoappネットワーク作成
```
$ docker container exec -it manager docker network create --driver=overlay --attachable todoapp
3j1wnc6fzh2niknr3e6ehusfg
```

* MySQLスタックをデプロイ
```
$ docker container exec -it manager docker stack deploy -c stack/todo-mysql.yml todo_mysql
Creating service todo_mysql_slave
Creating service todo_mysql_master
```

* MySQLコンテナに初期データ投入
```
$ docker container exec -it manager docker service ps todo_mysql_master --no-trunc --filter "desired-state=running" --format "docker container exec -it {{.Node}} docker container exec -it {{.Name}}.{{.ID}} bash"
docker container exec -it 9372f731e1c1 docker container exec -it todo_mysql_master.1.xbdp5ypk08ozt7ov4cna531ga bash

$ docker container exec -it 9372f731e1c1 docker container exec -it todo_mysql_master.1.xbdp5ypk08ozt7ov4cna531ga bash

$ docker container exec -it 9372f731e1c1 docker container exec -it todo_mysql_master.1.xbdp5ypk08ozt7ov4cna531ga init-data.sh

$ docker container exec -it 9372f731e1c1 docker container exec -it todo_mysql_master.1.xbdp5ypk08ozt7ov4cna531ga mysql -u gihyo -pgihyo tododb
mysql> SELECT * FROM todo LIMIT 1\G
*************************** 1. row ***************************
     id: 1
  title: MySQLのDockerイメージを作成する
content: MySQLのMaster、Slaveそれぞれで利用できるように、環境変数で役割を制御できるMySQLイメージを作成する
 status: DONE
created: 2020-11-01 17:50:03
updated: 2020-11-01 17:50:03
1 row in set (0.00 sec)

$ docker container exec -it manager docker service ps todo_mysql_slave --no-trunc --filter "desired-state=running" --format "docker container exec -it {{.Node}} docker container exec -it {{.Name}}.{{.ID}} bash"
docker container exec -it 3cb697256825 docker container exec -it todo_mysql_slave.1.4760xer3uihy2y4uu9x2odq75 bash
docker container exec -it 3a2de3f08dbe docker container exec -it todo_mysql_slave.2.khnlnjy7we87y9x84mw0q7jz4 bash

$ docker container exec -it 3cb697256825 docker container exec -it todo_mysql_slave.1.4760xer3uihy2y4uu9x2odq75 mysql -u gihyo -pgihyo tododb
mysql> SELECT * FROM todo LIMIT 1\G
*************************** 1. row ***************************
     id: 1
  title: MySQLのDockerイメージを作成する
content: MySQLのMaster、Slaveそれぞれで利用できるように、環境変数で役割を制御できるMySQLイメージを作成する
 status: DONE
created: 2020-11-01 17:50:03
updated: 2020-11-01 17:50:03
1 row in set (0.00 sec)
```

* todo_appスタックをデプロイ
```
$ docker container exec -it manager docker stack deploy -c stack/todo-app.yml todo_app
Creating service todo_app_nginx
Creating service todo_app_api

$ docker container exec -it manager docker service logs -f todo_app_api
todo_app_api.1.s1aw04ptkdkt@9372f731e1c1    | 2020/11/01 17:54:50 Listen HTTP Server
todo_app_api.2.2ail031y5c0k@3a2de3f08dbe    | 2020/11/01 17:54:50 Listen HTTP Server
```

* Nginxスタックをデプロイ(今回のメイン)
  * あれ、なんかだめっぽい。
```
$ docker container exec -it manager docker stack deploy -c stack/todo-app.yml todo_app
Updating service todo_app_api (id: kclcv92gfwdsjwjgt95xnubba)
Updating service todo_app_nginx (id: klx1x8racba49tdguglh0r2ur)
image registry:5000/ch04/nginx:latest could not be accessed on a registry to record
its digest. Each node will access registry:5000/ch04/nginx:latest independently,
possibly leading to different nodes running different
versions of the image.
```

* todo_app_nginxがうまく起動していないなー
```
$ docker container exec -it manager docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
kclcv92gfwds        todo_app_api        replicated          2/2                 registry:5000/ch04/todoapi:latest
klx1x8racba4        todo_app_nginx      replicated          0/2                 registry:5000/ch04/nginx:latest
nlhixxweyeil        todo_mysql_master   replicated          1/1                 registry:5000/ch04/tododb:latest
cdt3g4mmmo6a        todo_mysql_slave    replicated          2/2                 registry:5000/ch04/tododb:latest
```

* いったん削除。
```
$ docker container exec -it manager docker stack rm todo_app
Removing service todo_app_api
Removing service todo_app_nginx
```

* そうか今回の章では前回のtodo_appスタックを更新する形だから二重にデプロイしていたから変なエラーが出たのか。。これでよさそう。
```
$ docker container exec -it manager docker stack deploy -c stack/todo-app.yml todo_app
Creating service todo_app_api
Creating service todo_app_nginx
```

* ただ、todo_app_nginxのレプリカが0/2なのは気になる・・・ダメそうだが、、とりあえず今日はここまで
```
$ docker container exec -it manager docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                               PORTS
nz210v1bx7ar        todo_app_api        replicated          2/2                 registry:5000/ch04/todoapi:latest
0s0ivsgitps7        todo_app_nginx      replicated          0/2                 registry:5000/ch04/nginx:latest
nlhixxweyeil        todo_mysql_master   replicated          1/1                 registry:5000/ch04/tododb:latest
cdt3g4mmmo6a        todo_mysql_slave    replicated          2/2                 registry:5000/ch04/tododb:latest 
```

## 今日の学び
* Swarmクラスタを構築中にPCを再起動するとやり直しが大変・・・副作用として、改めてコマンド再確認できた。
* todo_app_nginxのレプリカは0/2なのは気になる・・・
* entrykitを使えばnginx.tmplからnginx.congを生成できる。tmplには環境変数を使える。
