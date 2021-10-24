---
title: "git blameができる拡張機能が欲しかったのでblamer.nvimを入れてみた"
date: 2021-10-24T23:09:15+09:00
draft: false
---

ファイルごとにgit blameができる拡張機能を探していたところ、下記記事をドンピシャで見つけたので入れてみました。  

[ポップアップウィンドウでgit blameを確認できる blamer\.nvim \- Qiita](https://qiita.com/Yoika/items/26553e8ad067b9e468e8)

blamer.nvimという名前ですが、popup機能が使える最新のVimであれば使うことができます。  

## git blameとは

> git blame の基本的な機能は、ファイルでコミットされた特定の行の作成者メタデータを表示することです。  
> [git blame \| Atlassian Git Tutorial](https://www.atlassian.com/ja/git/tutorials/inspecting-a-repository/git-blame)

## インストール方法

[APZelos/blamer\.nvim: A git blame plugin for neovim inspired by VS Code's GitLens plugin](https://github.com/APZelos/blamer.nvim#installation)

プラグイン管理にはvim-plugを使っています。  
vimrcに下記を追加して、`:PlugInstall`します。  

```vimrc
Plug 'APZelos/blamer.nvim'
```

## 設定

私はvimrcに次の設定をしました。  

```vimrc
" By default blamer_delay is 1000 ms
let g:blamer_delay = 500

let g:blamer_date_format = '%y/%m/%d'
```

また、`SPC`キーを起点としたキーマッピングで使えるようにvim-which-keyの設定も行っています。  

`SPC` + `g` + `b`で`:BlamerToggle`を実行します。  

```vimrc
" g is for git
let g:which_key_map.g = {
      \ 'name' : '+git' ,
      \ 'b' : [':BlamerToggle'      , 'git blame toggle'],
      \ }
```

## デモ

`:BlamerToggle`をすると、ノーマルモードで今いる行のgitのメタデータが表示されるようになります。  

また、`v`でビジュアルモードに切り替えて選択した複数行のgitのメタデータも表示されます。  

非常にシンプルな機能になっています。  

![demo](blamer-nvim.gif)

## まとめ

有名どころの[fugitive.vim](https://github.com/tpope/vim-fugitive)の機能の中に`:Git blame`という機能もありますが、  
git blameの機能のみが欲しかったためドンピシャでgit blame専用のプラグインが見つかって良かったです。  
