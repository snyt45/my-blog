---
title: "Docker/Kubernetes 実践コンテナ開発入門@24日目"
date: 2020-11-07T23:19:30+09:00
draft: false
---

[Docker/Kubernetes 実践コンテナ開発入門：書籍案内｜技術評論社](https://gihyo.jp/book/2018/978-4-297-10033-9)

昨日の記事が日付をまたいだ後に作ったため、日付的に同じですが今日は24日目になります。

今日はPodの章の続きからです。

## 5.7 ReplicaSet
* Podを定義したマニフェストファイルからは1つのPodしか作成できない
* 同じPodを複数生成・管理する場合はReplicaSetを利用する => Podでレプリカ数を設定できればよさそうだけど、そうじゃないみたい。

* simple-replicaset.yaml
  * ReplicaSetを定義したマニフェストファイル
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: echo
  labels:
    app: echo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: echo
  template: # template以下はPodリソースにおける定義と同じ
    metadata:
      labels:
        app: echo
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

* replicas
  * 作成するPod数を設定
* template
  * Pod定義と同じもの

* ReplicaSetがこの定義をもとにreplicas数だけPodの複製を行う

* ReplicaSetを反映
```
$ kubectl apply -f simple-replicaset.yaml
replicaset.apps/echo created
```

* Podの一覧を取得
  * Podが3つ作成されたことが確認できる
  * 同一のPod名が複製されるため、ランダムな識別子がサフィックスに付与されている
```
$ kubectl get pod
NAME         READY   STATUS    RESTARTS   AGE 
echo-hgdd6   2/2     Running   0          112s
echo-k2bgk   2/2     Running   0          112s
echo-rmfvn   2/2     Running   0          112s
```

* ReplicaSetを操作してPodの数を減らすと、減らした分のPodは削除される
* 削除されたPodは復元できない => ステートレスなPodに向いている

* ReplicaSetと関連するPodを削除
```
$ kubectl delete -f simple-replicaset.yaml
replicaset.apps "echo" deleted
```

## 5.8 Deployment

* Deployment
  * ReplicaSetを管理・操作するためのリソース
  * ReplicaSetより上位のリソース

* simple-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo
  labels:
    app: echo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: echo
  template: # template以下はPodリソースにおけるspec定義と同じ
    metadata:
      labels:
        app: echo
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

* Deploymentの定義はReplicaSetとほぼ差がない(差分はkindの設定のみ)
* 違いはDeploymentがReplicaSetの世代管理を可能にする点

* kubectlのコマンドを記録するために--recordオプションをつける
```
$ kubectl apply -f simple-deployment.yaml --record
deployment.apps/echo created
```

* 各リソース(pod、replicaset、deployment)を確認する
```
$ kubectl get pod,replicaset,deployment --selector app=echo
NAME                        READY   STATUS    RESTARTS   AGE
pod/echo-6968d87679-pxpbt   2/2     Running   0          41s
pod/echo-6968d87679-t5t2j   2/2     Running   0          41s
pod/echo-6968d87679-x2hjg   2/2     Running   0          41s

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/echo-6968d87679   3         3         3       41s

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/echo   3/3     3            3           41s
```

* Deploymentのリビジョンを確認
```
$ kubectl rollout history deployment echo
deployment.apps/echo 
REVISION  CHANGE-CAUSE
1         kubectl.exe apply --filename=simple-deployment.yaml --record=true
```

### 5.8.1 ReplicaSetライフサイクル

* k8sではDeploymentを1つの単位として、アプリケーションをデプロイする。
* 実運用ではReplicaSetを直接用いることは少ない

* Deploymentが管理するReplicaSet
  * 指定されたPod数の確保
  * 新しいバージョンのPodへの入れ替え
  * 以前のバージョンへのPodのロールバック

* Pod数のみを更新しても新規ReplicaSetは生まれない
* マニフェストファイルのreplicasを3から4に変更

```
$ kubectl apply -f simple-deployment.yaml --record
deployment.apps/echo configured
```

* 既存のPodがそのままで1つのコンテナが新たに作成されている
```
$ kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
echo-6968d87679-pxpbt   2/2     Running   0          60m
echo-6968d87679-t5t2j   2/2     Running   0          60m
echo-6968d87679-wzkvh   2/2     Running   0          38s
echo-6968d87679-x2hjg   2/2     Running   0          60m
```

* 新しくReplicaSetが生成されていればリビジョン番号2が表示されるはずですが、1のまま => replicasの変更ではReplicaSetの入れ替えが発生しないということ
```
$ kubectl rollout history deployment echo
deployment.apps/echo 
REVISION  CHANGE-CAUSE
1         kubectl.exe apply --filename=simple-deployment.yaml --record=true
```

* コンテナ定義を更新
* simple-deployment.yamlのechoコンテナのイメージを変更
```
- name: echo
  image: gihyodocker/echo:patched // ← 変更する
  ports:
  - containerPort: 8080
```

* Deploymentを反映
```
$ kubectl apply -f simple-deployment.yaml --record
deployment.apps/echo configured
```

* 新しいPodが作成され、古いPodが段階的に停止していく
```
$  kubectl get pod --selector app=echo
NAME                    READY   STATUS        RESTARTS   AGE
echo-54dbdb57c7-8jvfm   2/2     Running       0          49s
echo-54dbdb57c7-h6dkl   2/2     Running       0          50s
echo-54dbdb57c7-jxsh9   2/2     Running       0          40s
echo-54dbdb57c7-vmxfs   2/2     Running       0          36s
echo-6968d87679-pxpbt   0/2     Terminating   0          69m
echo-6968d87679-t5t2j   0/2     Terminating   0          69m
echo-6968d87679-x2hjg   2/2     Terminating   0          69m
```

* 最終的には新しいPodのみになる。
```
$  kubectl get pod --selector app=echo
NAME                    READY   STATUS    RESTARTS   AGE
echo-54dbdb57c7-8jvfm   2/2     Running   0          3m26s
echo-54dbdb57c7-h6dkl   2/2     Running   0          3m27s
echo-54dbdb57c7-jxsh9   2/2     Running   0          3m17s
echo-54dbdb57c7-vmxfs   2/2     Running   0          3m13s
```

* Deploymentのリビジョン確認
  * リビジョン2が作成されている。 => Deploymentの内容に変更があると新しいリビジョンが作成される
```
$ kubectl rollout history deployment echo
deployment.apps/echo 
REVISION  CHANGE-CAUSE
1         kubectl.exe apply --filename=simple-deployment.yaml --record=true
2         kubectl.exe apply --filename=simple-deployment.yaml --record=true
```

### 5.8.2 ロールバックを実行する
* Deploymentのリビジョンが記録されているため、特定のリビジョンの内容を確認できる

```
$ kubectl rollout history deployment echo --revision=1
deployment.apps/echo with revision #1
Pod Template:
  Labels:       app=echo
        pod-template-hash=6968d87679
  Annotations:  kubernetes.io/change-cause: kubectl.exe apply --filename=simple-deployment.yaml --record=true
  Containers:
   nginx:
    Image:      gihyodocker/nginx:latest
    Port:       80/TCP
    Host Port:  0/TCP
    Environment:
      BACKEND_HOST:     localhost:8080
    Mounts:     <none>
   echo:
    Image:      gihyodocker/echo:latest
    Port:       8080/TCP
    Host Port:  0/TCP
    Environment:        <none>
    Mounts:     <none>
  Volumes:      <none>
```

* undoを実行すれば直前の操作のリビジョンにDeploymentをロールバックができる
  * 最新のDeploymentに問題があった場合にすぐに以前のバージョンに戻すことができる
```
$ kubectl rollout undo deployment echo
deployment.apps/echo rolled back
```

* Deploymentと、関連するReplicastとPodの削除
```
$ kubectl delete -f simple-deployment.yaml
deployment.apps "echo" deleted
```

## 今日の学び
* k8sのマニフェストファイルはあくまでもコンテナをどう配置するのかということに集中できる。つまり、コンテナ自体をどういう構成にするかということはDockerイメージを作成時点に考慮しておく必要がある。
* kubectlのコマンド調べてた時によさげな翻訳サイトあったのでメモ
  * [Introduction · The Kubectl Book 日本語訳](https://kubectl-book-ja.netlify.app/)
* kubectl applyの-fと-kオプションの違いってなんだろう？？
* DeploymentはReplicaSetの上位のリソースでバージョン管理やロールバックが行えて便利。
