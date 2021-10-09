---
title: "Spacemacsのキーバインドが快適すぎたので同じ仕組みをvim-which-keyで導入する"
date: 2021-10-09T14:33:25+09:00
draft: false
---

## 少しだけ余談

皆さんはSpacemacsを知っていますか？  

[Spacemacs: Emacs advanced Kit focused on Evil](https://www.spacemacs.org/)

簡単に説明するとVimの操作とEmacsの操作の良いとこどりをしたエディタです。  
似たところでSpaceVimというプロジェクトもあります。  

元々Vimを触っていてEmacsのorg-modeなども触ってみたくてSpacemacsにも手を出したことがあります。  

テキスト編集で普通にVimのキーバインドが使えるし、Emacsの資産のorg-modeという素晴らしいモードも使えてほんとに良いとこどりのエディタでした。  

Spacemacsの実体はEmacsで、選りすぐりのプラグインや設定が施されたものが簡単に手に入るようになっているため、手順に従うだけですぐに使える状態のEmacsが手に入るような感覚でした。  

MRUなどもすでに搭載されていて設定しなくていいし、特に**スペースキーを活用したキーバインドが快適過ぎました。**

VimやEmacsを使う上でキーバインドを覚えるという高いハードルを軽減してくれました。  

**スペースキーを押すと、ポップアップで次のキーとそのキーを押すと何が起きるのかが書いてあるのでキーバインドを覚えずとも使えて最高な体験でした。**  

Spacemacsは一時マウス操作とキー操作を組み合わせて快適に使えていたのですが、やはり設定を変えようとするハードルは高かったのとSpacemacsはGUI
で使っていたのでWSL上で使おうとするとマウス操作？がうまく使えずに断念しました。  

## Vimにもvim-which-keyがあった
個人的にSpacemacsではEmacsの資産が使えるというのも大きかったですが、それ以上にスペースキーから始めるキーバインドが快適すぎたというのがあります。  

Vimでもないかなーと思って探していたら見つけました。  

[liuchengxu / vim\-which\-key：ポップアップにキーバインディングを表示するVimプラグイン](https://github.com/liuchengxu/vim-which-key)

説明にありますが、EmacsからVimに移植したものみたいです。  

> vim-which-key は emacs-which-key を vim に移植したもので、利用可能なキーバインディングを ポップアップで表示します。

## 実演
最初にvim-which-keyを入れるとこんな風になるよというデモです。  

Spacemacsのときは最初から色々設定されていたのですが、vim-which-keyは自分が設定したものだけ表示されてくれるのでメンテナンスは必要ですが、とても便利です！  

GIF動画では、スペースキーを押すと下にぴょこっとポップアップが出てきて、次に押せるボタンのキーバインドと説明が表示されます。  

今はウィンドウの垂直分割、水平分割、Fernを開くコマンドを割り当てています。  

![vim-which-keyデモ](vim-which-key.gif)

## 早速インストールする
プラグイン管理にはvim-plugを使っています。  

```vimrc
Plug 'liuchengxu/vim-which-key'
```

## 設定
ほんとにまだ最小限の設定しかしていないですが、プラグインを入れていくたびにメンテナンスしていこうと思います。  

スペースキー + eにFernのアクションを割り当てていたのですが、副作用が出たりする操作もありました。  
例えばペースト操作を行う際に同じファイルがあったら上書きするか、名前を変えるか入力を求めらるのですが、入力できなかったりなど。  
Fernはかなり使いやすく忘れても`?`などですぐヘルプ見れて困らないこともあり、一旦コメントアウトすることにしました。  

```vimrc
" By default timeoutlen is 1000 ms
set timeoutlen=500

" Map leader to which_key
nnoremap <silent> <leader> :<C-u>WhichKey '<Space>'<CR>
vnoremap <silent> <leader> :<C-u>WhichKeyVisual '<Space>'<CR>

" Create map to add keys to
let g:which_key_map = {}

" Single mappings
let g:which_key_map['-'] = [ '<C-W>s'  , 'split below']
let g:which_key_map['v'] = [ '<C-W>v'  , 'split right']
let g:which_key_map['e'] = [ ':Fern .' , 'explorer open']

" e is for explorer
" let g:which_key_map.e = {
"     \ 'name' : '+explorer' ,
"     \ 'h' : ['<Plug>(fern-action-help)'                 , 'help'],
"     \ 'o' : [':Fern .'                                  , 'open'],
"     \ 'C' : ['<Plug>(fern-action-clipboard-copy)'       , 'copy'],
"     \ 'D' : ['<Plug>(fern-action-remove)'               , 'remove'],
"     \ 'K' : ['<Plug>(fern-action-new-dir)'              , 'new dir'],
"     \ 'N' : ['<Plug>(fern-action-new-file)'             , 'new file'],
"     \ 'P' : ['<Plug>(fern-action-clipboard-paste)'      , 'paste'],
"     \ 'R' : ['<Plug>(fern-action-rename)'               , 'rename'],
"     \ 'X' : ['<Plug>(fern-action-clipboard-move)'       , 'cut'],
"     \ }

" Not a fan of floating windows for this
let g:which_key_use_floating_win = 0

" Hide status line
augroup vim-which-key
  autocmd!
  autocmd FileType which_key set laststatus=0 noshowmode noruler
augroup END

" Register which key map
call which_key#register('<Space>', "g:which_key_map")
```

また、設定をする際に下記を参考にしました。  

[fw/dotfiles \- \.config/nvim/keys/which\-key\.vim at 01b025e62ef5e28f1f14291a1bfe520a33d254aa \- dotfiles \- picture\-vision \- git service](https://git.picture-vision.com/fw/dotfiles/src/commit/01b025e62ef5e28f1f14291a1bfe520a33d254aa/.config/nvim/keys/which-key.vim)

## まとめ

Vimを使っているとプラグインと設定が増えていって数日経つと使わないキーバインドは忘れてしまって、結局使わなくなったりします。  
今回紹介したvim-which-keyもメンテナンスは必要ですが、メンテナンスをしていれば**とりあえずスペースキーを押せば何があったけと探すことができるようになるのでとても良いプラグインです。**  
