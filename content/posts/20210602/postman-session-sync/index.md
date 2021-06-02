---
title: "Postmanでログイン認証"
date: 2021-06-02T22:05:32+09:00
draft: false
---

Railsのバージョン：5.2

Railsのログイン認証画面にPostmanでPostしてログイン認証をやってみた。  
すると、`ActionController::InvalidAuthenticityToken`が出た。  

RailsのCSRF対策によるもの。  

## CSRFとは
Aさんがプロジェクト管理サービスにログイン  
↓  
ログアウトせずに、そのまま掲示板サイトを閲覧  
↓  
とある書き込みをブラウザが読み込む(`<img src="http://pj-service.com/project/1/destroy>`)  
↓  
ブラウザから上記URLに対してリクエストが走る(このとき、まだ有効なCookieが送信されるため認証後の画面でもリクエストが通ってしまう)  
↓  
Aさんが知らないうちにプロジェクトが削除されている。  

ブラウザが異なるドメインからのリクエストでも、そのドメインで利用できるCookieがある場合送信するというブラウザの仕様を突いたものみたいだ。  

[Rails セキュリティガイド \- Railsガイド](https://railsguides.jp/security.html#%E3%82%AF%E3%83%AD%E3%82%B9%E3%82%B5%E3%82%A4%E3%83%88%E3%83%AA%E3%82%AF%E3%82%A8%E3%82%B9%E3%83%88%E3%83%95%E3%82%A9%E3%83%BC%E3%82%B8%E3%82%A7%E3%83%AA-csrf)

## CSRF対策とは

> GET以外のリクエストにセキュリティトークンを追加することで、WebアプリケーションをCSRFから守ることができます。

[Rails セキュリティガイド \- Railsガイド](https://railsguides.jp/security.html#csrf%E3%81%B8%E3%81%AE%E5%AF%BE%E5%BF%9C%E7%AD%96)

下記をアプリケーションコントローラに設定することでCSRF対策が有効になる。  

```
protect_from_forgery with: :exception
```

ちなみにRails5.2以降ではデフォルトで有効になっているみたいで定義はされていなかった。  

> Rails5.2以降ではActionController::Base内で有効になっているため、デフォルトの設定でよければ自分で記述する必要はない

[RailsのCSRF対策について \- Qiita](https://qiita.com/eshow/items/915f8e8ad317aa8e49a6)

具体的には下記のことをやってみるみたいだ。  

> railsは、get以外の動詞のリンクに、authenticity_tokenというパラメータを自動的に付け加えます。get以外の動詞の各アクションでparams[:authenticity_token]とsession[:csrf_id]を比較して、同値であればOKとしているようです。*1同値でなければActionController::InvalidAuthenticityTokenという例外がでます。

[CSRFの対応について、rails使いが知っておくべきこと \- おもしろwebサービス開発日記](https://blog.willnet.in/entry/20080509/1210338845)

## CSRF対策が有効のままでどう認証を突破するか
本題です。  

PostmanでRailsの認証を突破する方法ですが、ググってもそもそもCSRF対策を無効にしようみたいな記事ばかりだったので今回自分でいろいろ試してみました。  

### その１：authenticity_token と ID と PasswordをPostmanから送信してみる

CSRFトークンはHTMLに埋め込まれているので、ログイン画面のCSRFトークンを取得。  
ブラウザのコンソールで下記を実行すると取得できます。  
HTML要素なので目視で探してもいいです。  
```
document.getElementsByName('csrf-token')[0].content
```

次にブラウザのコンソールを開いたまま画面から実際にログインしてみます。  

ネットワークタブから該当のアクセスを探して、Headers > Form Dataを見ます。  

authenticity_token  
user[login]  
user[password]  

が送信されています。

同じように上記のパラメータをPostmanからログイン画面にPostしてあげればよさそうですね。

ログアウトして、authenticity_tokenは新しいものを取得しましょう。

これで送信してみますがダメでした。

### その２：画面でログインして、ログイン後のセッションを使う

ブラウザのコンソールを開いたまま画面から実際にログインします。  

ネットワークタブからログイン後のページのアクセスを探して、Headers > Request Headers > Cookieを見ます。  

`_src_session`のキーと値をコピーします。  
PostmanのCookiesに追加します。  

この状態でログイン認証が必要なページにGETすると、なんと成功します。  

ログイン認証後のセッションがあれば問題なくログインできるようですね。  

### その３：ブラウザのセッション情報をPostmanと同期する

その２の方法でもよいですが、毎回面倒です。  

ブラウザのセッション情報をPostmanと同期する方法がありました。  

[PostmanでCookieが必要なAPIを実行する \- ユニファ開発者ブログ](https://tech.unifa-e.com/entry/2021/05/10/083532)

この方法なら、画面でログインするだけでPostmanに認証後のセッション情報が同期されているので、先ほどの面倒な手順なくリクエストを投げれば成功します。  

逆に画面でログアウトすれば認証が必要なリクエストの場合は失敗します。  

## まとめ

セッションさえ偽造できればログイン後の画面にアクセスできる。  
ブラウザは毎回送信できるCookieがあれば送信している。  
Postmanとブラウザのセッション情報を同期すれば認証後の画面へのリクエストが思いのままに投げられる。  

本来はAPIを投げる際はトークン認証とかなのですがそういった実装がなくとりあえずログイン認証後の画面のURLにリクエスト投げたい場合のTIPSでした。  
