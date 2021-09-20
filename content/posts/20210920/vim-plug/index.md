---
title: "Vim8のプラグイン管理にvim-plugを使う"
date: 2021-09-20T18:33:53+09:00
draft: false
---

最近少しずつVimの学びなおしをしています。  

下記の記事では、Vim8の標準パッケージ管理を使ってみました。  

[Vim8の標準パッケージ管理を使ってvim\-jaを入れてhelpを日本語化する • Small Changes](https://snyt45.com/posts/20210913/vim8-vim-ja/)

しかし、色々とプラグインを入れていこうと考えたとき**依存関係をいい感じに管理できない**なと思いvim-plugを入れることにしました。  

依存関係の管理といっても、**AのプラグインがBに依存していることが分かればいいです。**  
(homebrewのように勝手に依存関係を解決してくれると最高だけど…)

## vim-plugとは

[junegunn/vim\-plug: Minimalist Vim Plugin Manager](https://github.com/junegunn/vim-plug)

vim-plugを調べてみると、**導入が簡単でシンプルにプラグイン管理できるプラグインマネージャーという印象です。**  

## vim-plugの依存関係の書き方

[faq · junegunn/vim\-plug Wiki](https://github.com/junegunn/vim-plug/wiki/faq#managing-dependencies)

`|`のセパレータを使う方法と`手動インデント`を使う方法があるようです。  

私は`手動インデント`を使うのが視覚的に分かりやすいのでこれが決め手でvim-plugを使ってみようと思いました。  

```vimrc
" Vim script allows you to write multiple statements in a row using `|` separators
" But it's just a stylistic convention. If dependent plugins are written in a single line,
" it's easier to delete or comment out the line when you no longer need them.
Plug 'SirVer/ultisnips' | Plug 'honza/vim-snippets'
Plug 'junegunn/fzf', { 'do': './install --all' } | Plug 'junegunn/fzf.vim'

" Using manual indentation to express dependency
Plug 'kana/vim-textobj-user'
  Plug 'nelstrom/vim-textobj-rubyblock'
  Plug 'whatyouhide/vim-textobj-xmlattr'
  Plug 'reedes/vim-textobj-sentence'
```

## vim-plugを使ったプラグイン管理

### vim-plugのインストール

下記のコマンドを実行すると、`~/.vim/autoload/plug.vim`がインストールされます。  

```
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
    https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

### プラグインのインストール

`~/.vim/vimrc`を作成します。

```
touch ~/.vim/vimrc
```

`vimrc`に以下の設定を追加。

```vimrc
" use plugin manager from vim-plug
call plug#begin('~/.vim/plugged')
Plug 'vim-jp/vimdoc-ja'
Plug 'lambdalisue/fern.vim'
call plug#end()

" helpを日本語化
set helplang=ja,en
```

`vi`を再度開きなおして、`:Pluginstall`コマンドを実行するとプラグインがインストールされます。  

プラグインをインストールすると、`~/.vim/plugged`にプラグインがインストールされていました。  

### その他便利なコマンド

`:PlugUpdate`はプラグインを更新してくれるコマンドです。  

このようにプラグインを便利に管理できるコマンドも提供されます。  

[junegunn/vim\-plug: Minimalist Vim Plugin Manager](https://github.com/junegunn/vim-plug#commands)
