---
title: "Docker/Kubernetes 実践コンテナ開発入門@26日目"
date: 2020-11-09T22:32:14+09:00
draft: false
---

[Docker/Kubernetes 実践コンテナ開発入門：書籍案内｜技術評論社](https://gihyo.jp/book/2018/978-4-297-10033-9)

前回は、k8sのリソースのServiceについて学習しました。
Serviceは基本的にはPod→Podへの経路を指定するためのものでした。

今回はIngressについてやっていきます。

## 5.10 Ingress
* NodePortはL4層レベルまでのため、HTTP/HTTPSのようにL7層レベルの制御ができない
  * たまたま見つけたけど、層については下記の記事が参考になる
  * [『Docker/Kubernetes 実践コンテナ開発入門』輪読会 \#5 \- Speaker Deck](https://speakerdeck.com/nagaakihoshi/kubernetes-shi-jian-kontenakai-fa-ru-men-lun-du-hui-number-5?slide=18)
* これを解決するのがIngress
* 素のローカルk8s環境では使えない。

* そのため、nginx_ingress_controllerをデプロイします。
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.16.2/deploy/mandatory.yaml
namespace/ingress-nginx created
service/default-http-backend created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
unable to recognize "https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.16.2/deploy/mandatory.yaml": no matches for kind "Deployment" in version "extensions/v1beta1"
unable to recognize "https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.16.2/deploy/mandatory.yaml": no matches for kind "Deployment" in version "extensions/v1beta1"

$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.16.2/deploy/provider/cloud-generic.yaml
service/ingress-nginx created
```

* テキストが古いためかエラーが出ている。
  * no matches for kind "Deployment" in version "extensions/v1beta1"

* k8sのバージョンは1.18.8（DockerのSettingsのk8sの項目から確認できる）

* 1.18.8が対応するAPIを確認
  * `apps`を含むapiVersionのみがDeploymentに対して正しい
  * `extensions`はDeploymentをサポートしていない
  * [kubernetes — バージョン「extensions / v1beta1」の種類「Deployment」に一致するものはありません](https://www.it-swarm-ja.tech/ja/kubernetes/%E3%83%90%E3%83%BC%E3%82%B8%E3%83%A7%E3%83%B3%E3%80%8Cextensions-v1beta1%E3%80%8D%E3%81%AE%E7%A8%AE%E9%A1%9E%E3%80%8Cdeployment%E3%80%8D%E3%81%AB%E4%B8%80%E8%87%B4%E3%81%99%E3%82%8B%E3%82%82%E3%81%AE%E3%81%AF%E3%81%82%E3%82%8A%E3%81%BE%E3%81%9B%E3%82%93/813764018/)
```
$  kubectl api-resources | grep deployment
deployments                       deploy       apps                           true         Deployment
```

* 公式を見ながらやる方法もあるけど、、いったんテキストのyamlを書き換える方法でやってみる。
  * 公式
    * [マスターのingress\-nginx / index\.md・kubernetes / ingress\-nginx](https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/index.md)

* mandatory.yamlをローカルに落として、extensions/v1beta1をapps/v1に置換して再実行
* いけたー！！！
```
$ kubectl apply -f mandatory.yaml 
namespace/ingress-nginx created
deployment.apps/default-http-backend created
service/default-http-backend created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
deployment.apps/nginx-ingress-controller created
```

* さっきはPodが起動してなかったけど、ちゃんとPodも起動してくれた
* これでIngressリソースを利用する準備が整った
```
$ kubectl -n ingress-nginx get service,pod
NAME                           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
service/default-http-backend   ClusterIP      10.108.154.236   <none>        80/TCP                       93s
service/ingress-nginx          LoadBalancer   10.104.144.30    localhost     80:30719/TCP,443:31795/TCP   11s

NAME                                           READY   STATUS    RESTARTS   AGE
pod/default-http-backend-57fb4c77b4-zxq6b      1/1     Running   0          93s
pod/nginx-ingress-controller-7bfbcc484-ql2m8   1/1     Running   0          93s
```
### 5.10.1 Ingressを通じたアクセス
* Ingressを通してServiceにアクセス

* simple-service.yaml
  * spec.typeが未指定のときはClusterIP Serviceで作成される
```
apiVersion: v1
kind: Service
metadata:
  name: echo
spec:
  selector:
    app: echo
  ports:
    - name: http
      port: 80
```

* マニフェストファイルの変更を反映
```
$ kubectl apply -f simple-service.yaml
The Service "echo" is invalid: spec.ports[0].nodePort: Forbidden: may not be used when `type` is 'ClusterIP'
```

* エラーが出るので一旦Service削除
```
$ kubectl delete -f simple-service.yaml
service "echo" deleted
```

* 再度実行
```
$ kubectl apply -f simple-service.yaml
service/echo created
```

* simple-ingress.yamlを作成・反映
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: echo
spec:
  rules:
  - host: ch05.gihyo.local
    http:
      paths:
      - path: /
        backend:
          serviceName: echo
          servicePort: 80
```

```
$ kubectl apply -f simple-ingress.yaml
ingress.extensions/echo created
```

```
$ kubectl get ingress
NAME   CLASS    HOSTS              ADDRESS     PORTS   AGE
echo   <none>   ch05.gihyo.local   localhost   80      42s
```

* Ingeressのspec.rules.host属性に合致するのでバックエンドのServiceにリクエストが委譲されるようだ。

```
$ curl http://localhost -H 'Host: ch05.gihyo.local'
Hello Docker!!
```

* 他にもIngressの層では様々な制御ができる
* simple-ingress.yamlに次のような変更を加える

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: echo
  annotations:
    nginx.ingress.kubernetes.io/server-snippet: |
      set $agentflag 0;

      if ($http_user_agent ~* "(Mobile)" ){
        set $agentflag 1;
      }

      if ( $agentflag = 1 ) {
        return 301 http://gihyo.jp/;
      }

spec:
  rules:
  - host: ch05.gihyo.local
    http:
      paths:
      - path: /
        backend:
          serviceName: echo
          servicePort: 80
```

* 変更を反映する
```
$ kubectl apply -f simple-ingress.yaml
ingress.extensions/echo configured
```

* モバイルに偽装してリクエストを飛ばすとリダイレクトされることがわかる。
```
$ curl http://localhost -H 'Host: ch05.gihyo.local' -H 'User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 11_0 like Mac OS X) AppleWebKit/604.1.38 (KHTML, like Gecko) Version/11.0 Mobile/15A372 Safari/604.1'
<html>
<head><title>301 Moved Permanently</title></head>
<body bgcolor="white">
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx/1.13.12</center>
</body>$ curl http://localhost -H 'Host: ch05.gihyo.local' -H 'User-Agent: Mozilla/5.0 (iPhone; CPU iPhone OS 11_0 like Mac OS X) AppleWebKit/604.1.38 (KHTML, like Gecko) Version/11.0 Mobile/15A372 Safari/604.1'
<html>
<head><title>301 Moved Permanently</title></head>
<body bgcolor="white">
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx/1.13.12</center>
</body>
```

* バックエンドのWebサーバやアプリケーションがこのような制御から解放される
* IngressはパブリッククラウドにおいてはそのプラットフォームのL7ロードバランサーを利用可能
  * GCP
    * Cloud Load Balancing
  * AWS
    * Application Load Balancer

### コラム freshpodでイメージの更新を検知し，Podを自動更新する
* freshpod
  * イメージの更新を検知し、Podを自動更新する
  * webpackerのdev-serverとかに近いイメージを持った！

### コラム kube-prompt
* kube-prompt
  * macOS/Linux向け
  * kubectlのコマンドや、リソース名の候補を補完してくれるツール

### コラム Kubernetes API
* k8sのリソースの作成・更新・削除はk8sクラスタにデプロイされているAPIによって実行される
* apiVersionはリソースの操作に利用するAPIの種別
* k8sクラスタで利用できるAPIの確認方法
```
$ kubectl api-versions
```

* 各種リソースがどのAPIに対応しているかはリポジトリを見る。
  * https://github.com/kubernetes/api
* v1
  * Service、Pod
* apps/v1
  * Deployment
* extenxionx/v1beta1
  * Ingress

## 今日の学び
* Ingressは外部からのHTTP/HTTPSリクエストを受けることができ、Serviceに委譲することができる。
* 設定次第では、Webサーバやアプリケーション層のL7層の役割を担うことができる。
* 最初はテキスト通りにうまくいかずどうなるかと思ったけど、なんとか動いてくれてよかった！
