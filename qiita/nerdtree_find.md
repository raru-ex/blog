# 前置き

よくわからないなりにゴリ押しで設定しているため、何点か課題があります。
もっと良いやり方があったら教えてください。

# `NERDTreeFind`コマンドにハイライト機能を追加する

NERDTreeにはFindコマンドがあり、Treeの中にある現在開いているファイルにカーソルを瞬間移動させることができます。
このときついでにファイル名をハイライトしたいと思い、試行錯誤しつつ設定を追加してみました。


## 最終的な設定

※ 本題はfunction部分です。

```vim
nnoremap [nerd]   <Nop>
nmap <Space>n [nerd]

" NERDTreeTabsを利用するように変更
nnoremap <silent><C-e> :NERDTreeTabsToggle<CR>
nnoremap <silent>[nerd]f :call NERDTreeFindAndHighlight()<CR>
nnoremap <silent>[nerd]h :call NERDTreeHighlight()<CR>


" Findしつつファイルをハイライトする
function! NERDTreeFindAndHighlight()
  NERDTreeFind
  :setlocal isk+=.
  normal! 0w
  exe printf('match IncSearch /\<%s\>/', expand('<cword>'))
  :setlocal isk-=.
endfunction

" 開いてるファイルをハイライトする
function! NERDTreeHighlight()
  :call NERDTreeFindAndHighlight()
  :wincmd p
endfunction

```

## 補足・説明

- ハイライト箇所

```vim
normal! 0w
exe printf('match IncSearch /\<%s\>/', expand('<cword>'))
```

normalコマンドでnormalモードのキー操作を実行しています。
nerdtreeにカーソルが移動したときに必ずしも単語上にカーソルがないため、先頭に戻してから単語の箇所まで移動しています。

exeではcwordでカーソル下の単語を拾って、matchに喰わせています。
このときに、文字のハイライトを`IncSearch`の色で設定してます。[^1]


[^1]: 他にも[こんな色][link-1]とか[こんな色][link-2]が設定できるみたいです

- isKeywordの一時的な変更

```
:setlocal isk+=.
" 何かしらの処理
:setlocal isk-=.
```

vimではisKeywordという設定を見て単語の区切りを調整しています。
ここではglobalな変更は変えずにlocalなスコープで`.`を一連のキーワードとして認識するように設定しています。
この設定により`hogehoge.scala`のようなファイルを拡張子を含めて１単語として認識できるようになります。

## 課題
1. ファイル名先頭が.のものをハイライトできない
2. 操作ファイルを変更してもハイライトが残り続けて、煩わしくなる

# おわりに

大した記事ではありませんが、初めてvimでfunctionを作ったので記念に記録してみました。
細かいことは無視しているので、参考程度にどうぞ。


[link-1]:https://vim-jp.org/vimdoc-ja/syntax.html#group-name
[link-2]:https://vim-jp.org/vimdoc-ja/syntax.html#highlight-groups
