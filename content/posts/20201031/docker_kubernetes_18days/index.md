---
title: "Docker/Kubernetes 実践コンテナ開発入門@18日目"
date: 2020-10-31T23:47:12+09:00
draft: false
---

[Docker/Kubernetes 実践コンテナ開発入門：書籍案内｜技術評論社](https://gihyo.jp/book/2018/978-4-297-10033-9)

途中からですが、学習ログ取っていきます。

## 4.3 API Serviceの構築
* TODOアプリのドメインを担当するAPIの構築

```
$ git clone https://github.com/gihyodocker/todoapi
```

### 4.3.1 todoapiの基本構造
* アプリケーションの実行はcmd/main.goから始まる。
* windowsなのでtreeコマンドの表示が気に入らない…

```
$ tree /f todoapi

フォルダー パスの一覧:  ボリューム Windows
ボリューム シリアル番号は 6607-7B7E です
C:\USERS\SNYT45\DESKTOP\DOCKER_KUBERNETES 実践コンテナ開発入門\4-2\TODOAPI
│  .dockerignore
│  .gitignore
│  db.go
│  Dockerfile
│  env.go
│  go.mod
│  go.sum
│  handler.go
│
└─cmd
        main.go
```

* Docker関係ない。goの話。
* cmd/main.go
  * 環境変数の取得、MySQLへのDB接続処理、HTTPリクエストのハンドラ作成・エンドポイントの登録、サーバーの実行
```
package main

import (
	"fmt"
	"log"
	"net/http"
	"os"

	"github.com/gihyodocker/todoapi"
)

func main() {

	// (1) 必要な環境変数を格納した構造体を作成
	env, err := todoapi.CreateEnv()
	if err != nil {
		fmt.Fprint(os.Stderr, err.Error())
		os.Exit(1)
	}

	// (2) MySQL Masterへの接続するための構造体を作成
	masterDB, err := todoapi.CreateDbMap(env.MasterURL)
	if err != nil {
		fmt.Fprintf(os.Stderr, "%s is invalid database", env.MasterURL)
		return
	}

	// (3) MySQL Slaveへの接続するための構造体を作成
	slaveDB, err := todoapi.CreateDbMap(env.SlaveURL)
	if err != nil {
		fmt.Fprintf(os.Stderr, "%s is invalid database", env.SlaveURL)
		return
	}

	mux := http.NewServeMux()

	// (4) ヘルスチェック用APIのハンドラを作成
	hc := func(w http.ResponseWriter, r *http.Request) {
		log.Println("[GET] /hc")
		w.Write([]byte("OK"))
	}

	// (5) TODO操作APIのハンドラを作成
	todoHandler := todoapi.NewTodoHandler(masterDB, slaveDB)

	// (6) ハンドラをAPIエンドポイントとして登録
	mux.Handle("/todo", todoHandler)
	mux.HandleFunc("/hc", hc)

	// (7) サーバのポートやハンドラを設定し、Listenを開始
	s := http.Server{
		Addr:    env.Bind,
		Handler: mux,
	}
	log.Printf("Listen HTTP Server")
	if err := s.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}
```

### 4.3.2 アプリケーションでの環境変数の制御
* Docker関係ない。goの話。
* env.go
  * 環境変数を取得するための処理
  * cmd/main.goから呼びだす関数を定義
* CreateEnv関数
  * 必要な環境変数を取得してEnvという構造体に設定する
  * os.Getenvは環境変数を取得する関数
    * コンテナに設定した環境変数を取得

```
package todoapi

import (
	"errors"
	"os"
)

// 必要な環境変数を格納するための構造体
type Env struct {
	Bind      string
	MasterURL string
	SlaveURL  string
}

func CreateEnv() (*Env, error) {

	env := Env{}

	bind := os.Getenv("TODO_BIND") // APIをListenするポート設定
	if bind == "" {
		env.Bind = ":8080"
	}
	env.Bind = bind

	masterURL := os.Getenv("TODO_MASTER_URL") // MySQL Masterへの接続情報
	if masterURL == "" {
		return nil, errors.New("TODO_MASTER_URL is not specified")
	}
	env.MasterURL = masterURL

	slaveURL := os.Getenv("TODO_SLAVE_URL") // MySQL Slaveへの接続情報
	if slaveURL == "" {
		return nil, errors.New("TODO_SLAVE_URL is not specified")
	}
	env.SlaveURL = slaveURL

	return &env, nil
}
```

### 4.3.3 MySQL接続、テーブルマッピング
* Docker関係ない。goの話。
* db.go
  * MySQLに接続するための処理
* CreateDbMap関数
  * `[DBユーザー名]:[DBパスワード]@tcp([DBホスト]:[DBポート])/[DB名]`形式の接続情報を受け取る
  * MySQLとのコネクションを確立する
* Todo構造体
  * todoテーブルのマッピングするための構造体
  * SQLのSELECT文で取得されたレコードの値設定や、INSERT/UPDATE文を生成するのに利用
```
package todoapi

import (
	"database/sql"
	"time"

	_ "github.com/go-sql-driver/mysql"
	gorp "gopkg.in/gorp.v1"
)

func CreateDbMap(dbURL string) (*gorp.DbMap, error) {

	ds, err := createDatasource(dbURL)
	if err != nil {
		return nil, err
	}

	db := &gorp.DbMap{
		Db: ds,
		Dialect: gorp.MySQLDialect{
			Engine:   "InnoDB",
			Encoding: "utf8mb4",
		},
	}

	db.AddTableWithName(Todo{}, "todo").SetKeys(true, "ID")
	return db, nil
}

func createDatasource(dbURL string) (*sql.DB, error) {

	db, err := sql.Open("mysql", dbURL)
	if err != nil {
		return nil, err
	}

	db.SetMaxIdleConns(2)
	return db, nil
}

type Todo struct {
	ID      uint      `db:"id"      json:"id"`
	Title   string    `db:"title"   json:"title"`
	Content string    `db:"content" json:"content"`
	Status  string    `db:"status"  json:"status"`
	Created time.Time `db:"created" json:"created"`
	Updated time.Time `db:"updated" json:"updated"`
}
```

### 4.3.4 Handlerを実装する
* Docker関係ない。goの話。
* handler.go
  * HTTPリクエストを処理するためのHandler
  * TODOの操作に対応したHTTPリクエストに応じて、参照・作成、更新を行う。RailsでいうControllerのような存在。
  * cmd/main.goでNewTodoHandler関数で作成したHandlerを/todoというエンドポイントに設定。ServerHTTP関数がリクエストを受ける。

```
package todoapi

import (
	"database/sql"
	"encoding/json"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"time"

	gorp "gopkg.in/gorp.v1"
)

// Handlerの中でDBへクエリを発行するため、DB接続の構造体を持つ
type TodoHandler struct {
	master *gorp.DbMap
	slave  *gorp.DbMap
}

// Handlerの作成関数
func NewTodoHandler(master *gorp.DbMap, slave *gorp.DbMap) http.Handler {
	return &TodoHandler{
		master: master,
		slave:  slave,
	}
}

// HTTPリクエストを受け、ビジネスロジックを実行してレスポンスを返す
func (h TodoHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	log.Printf("[%s] RemoteAddr=%s\tUserAgent=%s", r.Method, r.RemoteAddr, r.Header.Get("User-Agent"))
	switch r.Method {
	case "GET":
		h.serveGET(w, r)
		return
	case "POST":
		h.servePOST(w, r)
		return
	case "PUT":
		h.servePUT(w, r)
		return
	default:
		NewErrorResponse(http.StatusMethodNotAllowed, fmt.Sprintf("%s is Unsupported method", r.Method)).Write(w)
		return
	}
}

type errorResponse struct {
	Status  int    `json:"status"`
	Message string `json:"message"`
}

func NewErrorResponse(status int, message string) *errorResponse {
	return &errorResponse{
		Status:  status,
		Message: message,
	}
}

func (e *errorResponse) Write(w http.ResponseWriter) {
	data, err := json.Marshal(e)
	if err != nil {
		log.Println("marshal error json is failed")
	}

	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(e.Status)
	w.Write(data)
}

func (h TodoHandler) serveGET(w http.ResponseWriter, r *http.Request) {

	status := r.URL.Query().Get("status")
	if status == "" {
		status = "TODO"
	}

	result, err := h.slave.Select(Todo{}, "SELECT * FROM todo WHERE status = ? ORDER BY updated DESC", status)
	if err != nil {
		log.Println(err.Error())
		NewErrorResponse(http.StatusInternalServerError, "Execute Query is failed").Write(w)
		return
	}

	todoItems := make([]*Todo, 0)

	for _, e := range result {
		todo := e.(*Todo)
		todoItems = append(todoItems, todo)
	}

	data, err := json.Marshal(todoItems)
	if err != nil {
		log.Println(err.Error())
		NewErrorResponse(http.StatusInternalServerError, "Marshal JSON is failed").Write(w)
		return
	}

	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.Write(data)
}

type TodoPostPayload struct {
	Title   string `json:"title"`
	Content string `json:"content"`
}

func (h TodoHandler) servePOST(w http.ResponseWriter, r *http.Request) {

	raw, err := ioutil.ReadAll(r.Body)
	if err != nil {
		log.Println(err.Error())
		NewErrorResponse(http.StatusInternalServerError, "Read payload is failed").Write(w)
		return
	}

	var payload TodoPostPayload
	if err := json.Unmarshal(raw, &payload); err != nil {
		log.Println(err.Error())
		NewErrorResponse(http.StatusInternalServerError, "Parse payload is failed").Write(w)
		return
	}

	now := time.Now()
	todo := Todo{
		Title:   payload.Title,
		Content: payload.Content,
		Status:  "TODO",
		Created: now,
		Updated: now,
	}

	if err := h.master.Insert(&todo); err != nil {
		log.Println(err.Error())
		NewErrorResponse(http.StatusInternalServerError, "Insert Data is failed").Write(w)
		return
	}

	w.Header().Set("Content-Type", "application/json; charset=utf-8")
	w.WriteHeader(http.StatusCreated)
}

type TodoPutPayload struct {
	ID      uint   `json:"id"`
	Title   string `json:"title"`
	Content string `json:"content"`
	Status  string `json:"status"`
}

func (h TodoHandler) servePUT(w http.ResponseWriter, r *http.Request) {

	raw, err := ioutil.ReadAll(r.Body)
	if err != nil {
		log.Println(err.Error())
		NewErrorResponse(http.StatusInternalServerError, "Read payload is failed").Write(w)
		return
	}

	var payload TodoPutPayload
	if err := json.Unmarshal(raw, &payload); err != nil {
		log.Println(err.Error())
		NewErrorResponse(http.StatusInternalServerError, "Parse payload is failed").Write(w)
		return
	}

	var target Todo
	if err := h.slave.SelectOne(&target, "SELECT * FROM todo WHERE id = ?", payload.ID); err != nil {
		if err == sql.ErrNoRows {
			NewErrorResponse(http.StatusNotFound, fmt.Sprintf("id=%d is not found", payload.ID)).Write(w)
		} else {
			log.Println(err.Error())
			NewErrorResponse(http.StatusInternalServerError, "Select todo is failed").Write(w)
		}
		return
	}

	now := time.Now()
	todo := Todo{
		ID:      payload.ID,
		Title:   payload.Title,
		Content: payload.Content,
		Status:  payload.Status,
		Created: target.Created,
		Updated: now,
	}

	if _, err := h.master.Update(&todo); err != nil {
		log.Println(err.Error())
		NewErrorResponse(http.StatusInternalServerError, "Update Data is failed").Write(w)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
}
```

### 4.3.5 APIのDockerfile
* Dockerfile
  * ベースイメージはgolang:1.13
  * WORKDIRで実行ディレクトリを指定
  * go getで依存ライブラリをインストール
  * todoapiディレクトリに移動&go build
  * 出来上がった実行ファイルのbin/todoapiを/usr/local/binにコピー
  * CMDでtodoapiを実行
```
FROM golang:1.13

WORKDIR /
ENV GOPATH /go

COPY . /go/src/github.com/gihyodocker/todoapi
RUN go get github.com/go-sql-driver/mysql
RUN go get gopkg.in/gorp.v1
RUN cd /go/src/github.com/gihyodocker/todoapi && go build -o bin/todoapi cmd/main.go
RUN cd /go/src/github.com/gihyodocker/todoapi && cp bin/todoapi /usr/local/bin/

CMD ["todoapi"]
```

* ch04/todoapi:latestというイメージ作成
```
$ docker image build -t ch04/todoapi:latest .
[+] Building 71.0s (11/11) FINISHED
 => [internal] load .dockerignore                                                                                                                                                0.1s 
 => => transferring context: 68B                                                                                                                                                 0.0s 
 => [internal] load build definition from Dockerfile                                                                                                                             0.1s 
 => => transferring dockerfile: 381B                                                                                                                                             0.0s 
 => [internal] load metadata for docker.io/library/golang:1.13                                                                                                                   3.3s 
 => [internal] load build context                                                                                                                                                0.0s 
 => => transferring context: 8.52kB                                                                                                                                              0.0s 
 => [1/7] FROM docker.io/library/golang:1.13@sha256:8ebb6d5a48deef738381b56b1d4cd33d99a5d608e0d03c5fe8dfa3f68d41a1f8                                                            52.5s 
 => => resolve docker.io/library/golang:1.13@sha256:8ebb6d5a48deef738381b56b1d4cd33d99a5d608e0d03c5fe8dfa3f68d41a1f8                                                             0.0s 
 => => sha256:d6ff36c9ec4822c9ff8953560f7ba41653b348a9c1136755e653575f58fbded7 50.40MB / 50.40MB                                                                                23.6s 
 => => sha256:c958d65b3090aefea91284d018b2a86530a3c8174b72616c4e76993c696a5797 7.81MB / 7.81MB                                                                                   4.1s 
 => => sha256:8ebb6d5a48deef738381b56b1d4cd33d99a5d608e0d03c5fe8dfa3f68d41a1f8 2.36kB / 2.36kB                                                                                   0.0s 
 => => sha256:24bd48a274920bf47ead96c5a2db8e6a3fbe26e8ae27557c2caa9aeae562a998 1.79kB / 1.79kB                                                                                   0.0s 
 => => sha256:d6f3656320fe38f736f0ebae2556d09bf3bde9d663ffc69b153494558aec9a79 6.19kB / 6.19kB                                                                                   0.0s 
 => => sha256:edaf0a6b092f5673ec05b40edb606ce58881b2f40494251117d31805225ef064 10.00MB / 10.00MB                                                                                 4.3s 
 => => sha256:80931cf6881673fd161a3fd73e8971fe4a569fd7fbb44e956d261ca58d97dfab 51.83MB / 51.83MB                                                                                28.7s 
 => => sha256:813643441356759e9202aeebde31d45192b5e5e6218cd8d2ad216304bf415551 68.67MB / 68.67MB                                                                                34.4s 
 => => sha256:799f41bb59c9731aba2de07a7b3d49d5bc5e3a57ac053779fc0e405d3aed0b9e 120.17MB / 120.17MB                                                                              49.4s 
 => => extracting sha256:d6ff36c9ec4822c9ff8953560f7ba41653b348a9c1136755e653575f58fbded7                                                                                        1.4s 
 => => extracting sha256:c958d65b3090aefea91284d018b2a86530a3c8174b72616c4e76993c696a5797                                                                                        0.2s 
 => => extracting sha256:edaf0a6b092f5673ec05b40edb606ce58881b2f40494251117d31805225ef064                                                                                        0.2s 
 => => extracting sha256:80931cf6881673fd161a3fd73e8971fe4a569fd7fbb44e956d261ca58d97dfab                                                                                        1.5s 
 => => sha256:16b5038bccc853e96f534bc85f4f737109ef37ad92d877b54f080a3c86b3cb3a 126B / 126B                                                                                      29.0s 
 => => extracting sha256:813643441356759e9202aeebde31d45192b5e5e6218cd8d2ad216304bf415551                                                                                        1.4s 
 => => extracting sha256:799f41bb59c9731aba2de07a7b3d49d5bc5e3a57ac053779fc0e405d3aed0b9e                                                                                        2.6s 
 => => extracting sha256:16b5038bccc853e96f534bc85f4f737109ef37ad92d877b54f080a3c86b3cb3a                                                                                        0.0s 
 => [2/7] COPY . /go/src/github.com/gihyodocker/todoapi                                                                                                                          2.2s 
 => [3/7] RUN go get github.com/go-sql-driver/mysql                                                                                                                              3.5s 
 => [4/7] RUN go get gopkg.in/gorp.v1                                                                                                                                            6.9s 
 => [5/7] RUN cd /go/src/github.com/gihyodocker/todoapi && go build -o bin/todoapi cmd/main.go                                                                                   2.0s 
 => [6/7] RUN cd /go/src/github.com/gihyodocker/todoapi && cp bin/todoapi /usr/local/bin/                                                                                        0.4s 
 => exporting to image                                                                                                                                                           0.1s 
 => => exporting layers                                                                                                                                                          0.1s 
 => => writing image sha256:cb5b230ef7bb2db9bfde1201e30a897a57ce5a4cebe2184275ca48f9b3ae40c1                                                                                     0.0s 
 => => naming to docker.io/ch04/todoapi:latest         
```

* localhost:5000/ch04/todoapi:latestとしてイメージをコピー
```
$ docker image tag ch04/todoapi:latest localhost:5000/ch04/todoapi:latest
```

* registryにイメージをpush
```
$ docker image push localhost:5000/ch04/todoapi:latest
The push refers to repository [localhost:5000/ch04/todoapi]
6c107bf012b7: Pushed
a83e8e7419b6: Pushed
9f89592b8d08: Pushed
60cf81dd6ed7: Pushed
2133a099401e: Pushed
ec7523d82c84: Pushed
73b60240f5be: Pushed
7279468fdfad: Pushed
e5df62d9b33a: Pushed
7a9460d53218: Pushed
b2765ac0333a: Pushed
0ced13fcf944: Pushed
latest: digest: sha256:4cb1de36df9abf748c7c35c49109249f2db91237257360b3f0e123f298462227 size: 2847
```

### 4.3.6 Swarm上でtodoapiサービスを実行する
* stackディレクトリにtodo-app.yml作成

```
version: "3"
services:
  api:
    image: registry:5000/ch04/todoapi:latest
    deploy:
      replicas: 2
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

* todo_appというStack名でデプロイ
```
$ docker container exec -it manager docker stack deploy -c stack/todo-app.yml todo_app
Creating service todo_app_api
```

* docker service logs -fでtodo_app_apiが正常に起動しているか確認
  * 「Listen HTTP Server」が表示されていればAPIサーバとしてリクエストを受けられる状態
```
$ docker container exec -it manager docker service logs -f todo_app_api
todo_app_api.2.nuymt58kziy1@095bc5ce9b86    | 2020/10/31 17:14:45 Listen HTTP Server
todo_app_api.1.ztwpncv4aqg1@38cac483943e    | 2020/10/31 17:14:45 Listen HTTP Server
```

## 今日の学び
* 今回はほぼgoの話。
* Docker関係ないけど、go言語のみFWなしでtodoapiが作れるのが凄い。あと、ファイル数少なくてうらやましい(FWじゃないから当たり前か)。
* docker service logsでサービスで稼働しているコンテナのログが確認できる。
