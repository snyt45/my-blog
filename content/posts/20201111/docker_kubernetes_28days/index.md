---
title: "DDocker/Kubernetes 実践コンテナ開発入門@28日目"
date: 2020-11-11T22:54:20+09:00
draft: false
---

[Docker/Kubernetes 実践コンテナ開発入門：書籍案内｜技術評論社](https://gihyo.jp/book/2018/978-4-297-10033-9)

前回はGCP上でk8sクラスタを作成するための導入でした。

6章ではTODOアプリを公開していきますが、今回はその構成のうちのMySQLを構築していきます。

## 6.2 GKE上にTODOアプリケーションを構築する
* k8sのリソースでDockerをラップしてGKE上にTODOアプリケーションを構築
* MySQL → API → Webアプリケーションの順でデプロイ。最終的にIngressで公開

## 6.3 Master Slave構成のMySQLをGKE上に構築する
* k8sでは、ホストから分離可能な外部ストレージをボリュームとして利用
* この仕組みを実現するk8sのリソース
  * PersistentVolume
  * PersistentVolumeClaim
  * StorageClass
  * StatefulSet

### 6.3.1 PersistentVolumeとPersistentVolumeClaim
* PersistentVolume
  * ストレージの実体
* PersistentVolumeClaim
  * PersistentVolumeに対して必要な容量を動的に確保

* PersistentVolumeClaimのマニフェストファイルのイメージ
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ssd // ← 実体はStorageClassリソース
  resources:
    requests:
      storage: 4Gi
```
* accessModes
  * Podからストレージへのマウントポリシー
  * ReadWriteOnceは1つのノードからのR/Wマウントのみ許可
* srorageClassName
  * 利用するストレージの種類
* resources.requests.storage
  * 必要な容量を指定

### 6.3.2 StorageClass
* StorageClass
  * PersistentVolumeが確保するストレージの種類を定義
  * 直前のPersistentVolumeClaimのstorageClassNameで指定した値の実体
* GCPのストレージ
  * 「標準」と「SSD」が存在
* storage-class-ssd.yaml
```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
  labels:
    kubernetes.io/cluster-service: "true"
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```
* SSDに対応
* provisioner
  * GCEPersistentDiskに対応したVolumePluginのgce-pdを指定
* parameters
  * gce-pdのパラメータのtype属性にSSDに対応したpd-ssdを指定

* StorageClassを作成
```
$ kubectl apply -f storage-class-ssd.yaml
storageclass.storage.k8s.io/ssd created
```

### 6.3.3 StatefulSet
* StatefulSet
  * データを永続化するステートフルなアプリケーションの管理に向いたリソース
  * Podにpod-0、pod-1、pod-2のような連番で一意な識別子でPodが作成される
  * 識別子はPodが再作成されても保たれる

* mysql-master.yaml
  * Masterの設定
```
apiVersion: v1
kind: Service
metadata:
  name: mysql-master
  labels:
    app: mysql-master
spec:
  ports:
  - port: 3306
    name: mysql
  clusterIP: None
  selector:
    app: mysql-master // Serviceを介してStatefulSetのmysql-masterにアクセス
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-master
  labels:
    app: mysql-master
spec:
  serviceName: "mysql-master"
  selector:
    matchLabels:
      app: mysql-master
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql-master
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: mysql
        image: gihyodocker/tododb:latest
        imagePullPolicy: Always
        args:
        - "--ignore-db-dir=lost+found"
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "gihyo"
        - name: MYSQL_DATABASE
          value: "tododb"
        - name: MYSQL_USER
          value: "gihyo"
        - name: MYSQL_PASSWORD
          value: "gihyo"
        - name: MYSQL_MASTER
          value: "true"
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates: // ← ReplicaSetと違う点
  - metadata:
      name: mysql-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: ssd
      resources:
        requests:
          storage: 4Gi
```

* mysql-masterのServiceとStatefulSetを作成
```
$ kubectl apply -f mysql-master.yaml
service/mysql-master unchanged
statefulset.apps/mysql-master created
```

* StatefulSetはステートフルなReplicaSetという位置づけ
* volumeClaimTemplate
  * PersistentVolumeClaimをPodごとに自動生成するためのテンプレートを定義できる

* mysql-slave.yaml
  * Slaveの設定
```
apiVersion: v1
kind: Service
metadata:
  name: mysql-slave
  labels:
    app: mysql-slave
spec:
  ports:
  - port: 3306
    name: mysql
  clusterIP: None
  selector:
    app: mysql-slave

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-slave
  labels:
    app: mysql-slave
spec:
  serviceName: "mysql-slave"
  selector:
    matchLabels:
      app: mysql-slave
  replicas: 2
  updateStrategy:
    type: OnDelete
  template:
    metadata:
      labels:
        app: mysql-slave
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: mysql
        image: gihyodocker/tododb:latest
        imagePullPolicy: Always
        args:
        - "--ignore-db-dir=lost+found"
        ports:
        - containerPort: 3306
        env:
        - name: MYSQL_MASTER_HOST // ← Masterの場所を知る必要があるため、MasterのService名を指定
          value: "mysql-master"
        - name: MYSQL_ROOT_PASSWORD
          value: "gihyo"
        - name: MYSQL_DATABASE
          value: "tododb"
        - name: MYSQL_USER
          value: "gihyo"
        - name: MYSQL_PASSWORD
          value: "gihyo"
        - name: MYSQL_REPL_USER
          value: "repl"
        - name: MYSQL_REPL_PASSWORD
          value: "gihyo"
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: ssd
      resources:
        requests:
        storage: 4Gi
```

* slaveのServiceとStatefulSetを作成
```
$ kubectl apply -f mysql-slave.yaml
service/mysql-slave created
statefulset.apps/mysql-slave created
```

* 作成されたPodの確認
```
$  kubectl get pod
NAME             READY   STATUS    RESTARTS   AGE
mysql-master-0   1/1     Running   1          14m
mysql-slave-0    1/1     Running   0          117s
mysql-slave-1    1/1     Running   0          99s
```

* 初期データを登録
```
$  kubectl exec -it mysql-master-0 init-data.sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
```

* Podに反映されているか確認
```
$  kubectl exec -it mysql-slave-0 bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
root@mysql-slave-0:/#  mysql -u root -pgihyo tododb -e "SHOW TABLES;"
mysql: [Warning] Using a password on the command line interface can be insecure.
+------------------+
| Tables_in_tododb |
+------------------+
| todo             |
+------------------+
```

## 今日の学び
* 外部からPodにアクセスできるようにServiceを作成
* StatefuleSetを使ってmasterとslaveのPodを作成
* StatefulSetの中でvolumeClaimTemplatesを定義でき、ボリュームの容量やストレージタイプを定義。volumeMountsでvolumeClaimTemplatesで作成したボリュームを指定することでデータを永続化することができる。
