---
title: "Docker/Kubernetes 実践コンテナ開発入門@25日目"
date: 2020-11-08T23:15:46+09:00
draft: false
---

[Docker/Kubernetes 実践コンテナ開発入門：書籍案内｜技術評論社](https://gihyo.jp/book/2018/978-4-297-10033-9)

昨日はReplicaSetでPodの複製とその上位のDeploymentでReplicaSetのバージョン管理やロールバックを行いました。

今回はServiceについてやっていきます。

## 5.9 Service
* Serviceとは、Podの集合にアクセスするための経路を定義するリソース
* Serviceのターゲットとなる一連のPodは、Serviceで定義するラベルセレクタによって決定

* simple-replicaset-with-label.yaml
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo-spring
  labels:
    app: echo
    release: spring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: echo
      release: spring
  template:
    metadata:
      labels:
        app: echo
        release: spring
    spec:
      containers:
      - name: nginx
        image: gihyodocker/nginx:latest
        env:
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80
      - name: echo
        image: gihyodocker/echo:latest
        ports:
        - containerPort: 8080

---
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo-summer
  labels:
    app: echo
    release: summer
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo
      release: summer
  template:
    metadata:
      labels:
        app: echo
        release: summer
    spec:
      containers:
      - name: nginx
        image: gihyodocker/nginx:latest
        env:
        - name: BACKEND_HOST
          value: localhost:8080
        ports:
        - containerPort: 80
      - name: echo
        image: gihyodocker/echo:latest
        ports:
        - containerPort: 8080
```

* マニフェストファイルをk8sクラスタに反映
```
$ kubectl apply -f simple-replicaset-with-label.yaml
replicaset.apps/echo-spring created
replicaset.apps/echo-summer created
```

* 作成されたPodを確認
```
$ kubectl get pod -l app=echo -l release=spring
NAME                READY   STATUS    RESTARTS   AGE
echo-spring-5dprx   2/2     Running   0          55s
```

```
$ kubectl get pod -l app=echo -l release=summer
NAME                READY   STATUS    RESTARTS   AGE
echo-summer-2xj8q   2/2     Running   0          2m47s
echo-summer-vjv9j   2/2     Running   0          2m47s
```

* release=summerを持つPodだけにアクセスできるServiceを作る

* simple-service.yaml
  * spec.selector属性には、ServiceのターゲットとなるPodが持つラベルを指定
    * 対象のPodがあった場合はService経由でトラフィックが流れるようになる => Railsでいうルーティングに近いものがある
```
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector:
    app: echo
    release: summer
  ports:
    - name: http
      port: 80
```

* k8sクラスタに反映
```
$ kubectl apply -f simple-service.yaml
service/echo created
```

* 作成されたServiceを確認
```
$  kubectl get svc echo
NAME   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
echo   ClusterIP   10.111.36.1   <none>        80/TCP    20s
```

* 実際に、release=summerを持つPodだけにトラフィックが流れる確認
* 基本的にServiceはk8sクラスタの中からしかアクセスできない => 外部からはアクセスできないということか

* k8sクラスタ内にデバッグコンテナをデプロイ → デバッグコンテナからcurlコマンドでHTTPリクエストを送る
```
$ kubectl run -i --rm --tty debug --image=gihyodocker/fundamental:0.1.0 --restart=Never -- bash -il
If you don't see a command prompt, try pressing enter.
debug:/# curl http://echo/ # ← HTTPリクエストを送信
Hello Docker!!debug:/# 
```

* 各Podのログを確認すると、summerのラベルを持つPodのみログが確認できる
```
$ kubectl logs -f echo-summer-2xj8q -c echo
2020/11/08 14:35:35 start server

$ kubectl logs -f echo-summer-vjv9j -c echo
2020/11/08 14:35:32 start server
2020/11/08 14:48:14 received request # ← summerのラベルを持つPodのみログが出力されている

$ kubectl logs -f echo-spring-5dprx -c echo
2020/11/08 14:35:28 start server
```

### コラム Serviceの名前解決

* k8sクラスタ内のDNSでは、Serviceを`Service名.Namespace名.svc.local`で名前解決できる

```
$ curl http://echo.default.svc.local # 基本はこう

$ curl http://echo.default # svc.lodalは省略可能

$ curl http://echo # さらに同一Namespace内であればNamespace名も省略可能
```

### 5.9.1 ClusterIP Service
* Seviceには様々な種類がある。
* デフォルトはClusterIP Service
* K8sクラスタ上の内部IPアドレスにServiceを公開できる
* Podから別のPodへのアクセスがServiceを介してできる => ただし、外からはアクセスできない

### 5.9.2 NodePort Service
* クラスタ外からアクセスできるService
* 各ノード上からServiceポートへ接続するためのグローバルなポートを開ける

```
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  type: NodePort
  selector:
    app: echo
  ports:
    - name: http
      port: 80
```

* Serviceを作成
```
$ kubectl apply -f simple-node-port-service.yaml
service/echo configured
```

* 作成したServiceを確認
  * 80:30026/TCPと表示されているようにノードの30026ポートからServiceにアクセスできる
```
$ kubectl get svc echo
NAME   TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
echo   NodePort   10.111.36.1   <none>        80:30026/TCP   25mj
```

* 次のようにhttp://127.0.0.1:30026でアクセスできる
```
$ curl http://127.0.0.1:30026
Hello Docker!!
```

### 5.9.3 LoadBalancer Service
* ローカルk8s環境では利用できないService
* 主に各クラウドプラットフォームで提供されているロードバランサーと連携するためのもの
  * GCP
    * Cloud Load Balancingが対応
  * AWS
    * Elastic Load Balancingが対応

### 5.9.4 ExternalName Service
* selectorもport定義もない特殊なService
* k8sクラスタ内から外部のホストを解決するためのエイリアスを提供

* 次の例では、gihyo.jpをgihyouで名前解決できる
```
apiVersion: v1
kind: Service
metadata:
  name: gihyo
spec:
  type: ExternalName
  externalName: gihyo.jp
```

## 今日の学び
* Serviceは、アクセスする先のPodを指定するためのリソース
  * spec.selector属性で、ServiceのターゲットとなるPodが持つラベルを指定
* Serviceには様々な種類がある。
  * ClusterIP Service
    * Pod → Service → Pod ※外からはアクセス不可
  * NodePort Service
    * 外 → Serveic → Pod
  * LoadBalancer Service
    * ロードバランサーと連携
  * ExternalName Service
    * 名前解決するためのService
