---
title: "Windows TerminalのVimのカーソル形状をカスタマイズする"
date: 2021-11-19T22:00:19+09:00
draft: false
---

Windows Terminalからポインターの設定ができますが、  
Windows Terminalから設定を行った場合は一律で指定されたポインターの形状になってしまいます。  

Vimを使う際は、  

- ノーマルモードのときは、ブロック
- インサートモードのときは、縦棒

の形状にしたいです。  

こちらの記事を参考に設定してみると、Windows Terminalでも意図した通りのカーソル形状になりました！  

[端末の Vim でも挿入モードで縦棒カーソルを使いたい \- Qiita](https://qiita.com/Linda_pp/items/9e0c94eb82b18071db34)

```vimrc
if has('vim_starting')
    " 挿入モード時に非点滅の縦棒タイプのカーソル
    let &t_SI .= "\e[6 q"
    " ノーマルモード時に非点滅のブロックタイプのカーソル
    let &t_EI .= "\e[2 q"
    " 置換モード時に非点滅の下線タイプのカーソル
    let &t_SR .= "\e[4 q"
endif
```
