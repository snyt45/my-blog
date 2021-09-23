# TechDiary
Sites created with Hugo

## 環境構築
```
# Hugoをインストール
brew install hugo

# Hugoのバージョン確認
hugo version

# プロジェクトをクローン ※--recursiveでサブモジュールも同時にクローン
git clone --recursive https://github.com/snyt45/my-blog.git ~/tmp/my-blog
```

## 運用
### ローカルでの記事確認
１． 記事作成
```
hugo new posts/20200627/test_post/index.md
```

２． ビルド
```
hugo
```

３． サーバー起動
```
hugo server
```

４． 生成されたURLにアクセス

### デプロイ
ホスティング先をNetlifyにしている｡
リモートにプッシュすると､自動でデプロイしてくれる｡

## その他
### 画像管理方法
https://snyt45.com/posts/20200627/hugo_post_with_add_images/

画像パスの記載方法：
```
![代替テキスト](./image.jpg)
```

### ツイートの埋め込み
参考：https://foresuke.com/post/hugo_embed/

### asciinemaの埋め込み
参考：https://jenciso.github.io/blog/embedding-asciinema-cast-in-your-hugo-site/
※asciinema.htmlのsrc箇所は最初の`/casts/`を削除する必要あり。

### 改行
文末尾に半角スペース2つ
