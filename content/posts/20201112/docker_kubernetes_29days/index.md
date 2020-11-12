---
title: "Docker/Kubernetes 実践コンテナ開発入門@29日目"
date: 2020-11-12T22:10:51+09:00
draft: false
---

[Docker/Kubernetes 実践コンテナ開発入門：書籍案内｜技術評論社](https://gihyo.jp/book/2018/978-4-297-10033-9)

前回は、主にStatefulSetを使ったMySQLアプリケーションをk8sにデプロイしました。

今回はTODO APIとWebアプリケーションをデプロイしていきます。

## 6.4 TODO APIをGKE上に構築する

* todo-api.yaml
```
// Serviceは作成されると一意のIPアドレスが割り当てられる
// PodはServiceと通信するように構成でき、Serviceへの通信はServiceのメンバーであるPodに自動的に負荷分散できる
apiVersion: v1
kind: Service
metadata:
  name: todoapi // Serviceの名前
  labels:
    app: todoapi // Serviceのラベル
spec: // Serviceの仕様
  selector:
    app: todoapi // app: todoapiラベルを持つ任意のPodのhttpポート80をターゲットとするサービスを作成
  ports:
    - name: http
      port: 80 // デフォルトではtargetPortフィールドはportフィールドと同じ値で設定される
               // portはServiceのポート => 他のPodがサービスにアクセスする際に使う
               // targetPortはコンテナがトラフィックを受信するポート

---
apiVersion: apps/v1
kind: Deployment // ReplicaSetを管理するリソース
metadata:
  name: todoapi // Deploymentの名前
  labels:
    app: todoapi // Deploymentのラベル
spec:
  replicas: 2 // レプリカ数の指定 nginxとapiのPodがそれぞれ2つ作成される
  selector: // Label Selector
    matchLabels: // 対象とするPodのラベルを指定
      app: todoapi
  template: // Pod Template
    metadata:
      labels:
        app: todoapi // Deploymentの対象となるPod
    spec:
      containers: // 今回のPodはnginxとapiで構成される
      - name: nginx
        image: gihyodocker/nginx:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        env:
        - name: WORKER_PROCESSES
          value: "2"
        - name: WORKER_CONNECTIONS
          value: "1024"
        - name: LOG_STDOUT
          value: "true"
        - name: BACKEND_HOST
          value: "localhost:8080" // APIは同一Pod内に存在するのでlocalhostで名前解決できる
      - name: api
        image: gihyodocker/todoapi:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
        env:
        - name: TODO_BIND
          value: ":8080"
        - name: TODO_MASTER_URL
          value: "gihyo:gihyo@tcp(mysql-master:3306)/tododb?parseTime=true"
        - name: TODO_SLAVE_URL
          value: "gihyo:gihyo@tcp(mysql-slave:3306)/tododb?parseTime=true"
```

* APIアプリケーションを作成
```
$  kubectl apply -f todo-api.yaml
service/todoapi created
deployment.apps/todoapi created
```

* Podが正常に作成されていることを確認
```
$  kubectl get pod -l app=todoapi
NAME                       READY   STATUS    RESTARTS   AGE
todoapi-7758d95d4b-kqxpk   2/2     Running   0          61m
todoapi-7758d95d4b-z9ggc   2/2     Running   0          61m
```

## 今日の学び
* リソースが出てくるけど、リソースの概念の大きさとかをイメージできると思い出しやすい。例えば、DeploymentはReplicaSetの上位互換とか、ReplicaSetはPodの集合体とか。
* 今までちゃんとマニフェストファイルの書き方とか意味とか調べてなかったけど、kubectl explainが便利。Deploymentのmetadataの書き方が知りたかったらkubectl explain deployment.metadataで説明が出てくる。
* また一気に分からない所が出てきて調べてなかったけど、その時点で少しずつ調べながら不足している部分を補填していかないとどんどん分からなくなる。今回はわかる範囲でyamlファイルにコメントした。
