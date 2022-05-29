---
title: "nvimとVSCodeを必要に応じて使い分けられるような自分なりの設定"
emoji: "✌️"
type: "tech"
topics: ["nvim", "vscode"]
published: false
---

こんにちは。
nvimとVSCodeを行ったりきたりしているbunです。

どちらのエディターもライトユーザー感丸出しですが、vimとVSCodeを両方使っている方は意外といらっしゃるのではないかと思います。

私の場合は

- 基本的にコーディングやメモはnvimで行いたい
- ペアプロやVSCodeの拡張機能が利用したい時にはVSCodeを違和感なく利用できるようにしたい
  - その他古いFWを使っているプロジェクトでVSCode用のフォーマッターは用意できたがvimでやろうとするのは面倒な場合など

といった感じで使い分けたいと考えています。

どっちの良いところも享受できれば嬉しいなと。

以前は [VSCodeのnvim拡張を使っていた](https://zenn.dev/bun913/articles/02785aed0ba50e)のですが、どうしても怪しい挙動をすることも多く、nvimのさくさく感も味わいたいので、 `基本ターミナルに住むようにして、必要に応じてGUIの世界に戻ろう` と決めたのが1週間前くらいの出来事です。

ただ VSCodeを使う時にもnvimを使う時にも似たような開発体験をするためには、各種プラグインの導入・キーバインドの設定などが必要であるため、1週間くらいかけて諸々抜本的なnvim環境の作成しました。


同僚「**そんなことするくらいならVSCodeでいいじゃん・・・**」

僕「**こんなにかっこよく育ってくれたターミナルとnvimの前でよくそんなこと言えるな！！この子の前でもう一度言ってみろ！！**」

![コーディング風景](/images/nvim/terminal.jpeg)

同僚(クソデカため息)「**そんなことするくらいならVSCodeでいいじゃん・・・**」

というやりとりもあったとか・・・

## 概要・前提条件など

以下のような意図でnvim等の設定を行いました。

- 全てのキーバインドを2つのエディタで合わせるのではなく、タブの移動とかファイラーの操作感などの基本操作を似せて、違和感なく両方のエディターを使えるようにする
- 双方のエディターに便利な機能があるので、得意なことを得意な方にやってもらう

また、筆者は弱々vimmerですので、以下に記載する内容より良い方法・設定があると思いますのでご容赦ください。


【筆者の環境】

- macOS BigSur(AppleSiliconとIntelチップ両方で動作確認済み)


## コーディングの基本道具・基本動作

基本的なvimの操作方法などはここでは割愛しております。

開発において似たような操作感にするためにした設定内容・実際にどのように動かすのかご紹介いたします。


### 基本道具

### ターミナル

ターミナルとして Macに weztermを使用しています。

https://github.com/wez/wezterm

yutkat さんがzennでも紹介されておりましたので、そちらを見て導入しました。

https://zenn.dev/yutakatay/articles/wezterm-intro

私は主にtmuxを利用しますので、正直 alacrittyでもよかったのですが、何も考えずに `brew` でインストールした時に、日本語のインライン入力ができなかったので、諦めました。

正直 weztermで `iTerm2` より体感速度が上昇したので、十分かなぁなんて思っています。

### nvim

nvim の version 0.7.0を利用しています。

https://github.com/neovim/neovim

以下、nvimをvimと表現します。また、私は `vim` を `nvim` の aliasとして登録しておりますので、 `vim` というコマンドで `nvim` を起動しております。

### 基本動作

まずはターミナルで tmuxを起動します。

その次にお気に入りのペインに分割するためのシェルの関数を呼び出します。

私は小さい画面では `mini_ide` 、大きい画面では `ide` と呼び出す関数を使い分けたりしています。

そして tmuxで分割した画面の移動は基本的に `ctrl` を押しながら

- `h` で左方向
- `l` で右方向
- `k` で上方向
- `j` で下方向

に移動できるようにキーを設定しております。

![tmuxのペイン移動](/images/nvim/tmux_movement.gif)

また、vimで分割した画面についても同様のキーで移動できるようにしております。

![vimのペイン移動](/images/nvim/vim_movement.gif)

また、後述する `vim-tmux-navigator` という vimの拡張機能を利用して vimで分割したペイン、tmuxで分割したペイン双方の移動も同様に `ctrl` キーで移動できるようにしています。

![vim<->tmuxペイン移動](/images/nvim/vim_tmux_movement.gif)

私はVSCodeで `cmd+l` で右のタブに、 `cmd+h` で左のタブに移動するように設定しております。

```json
// VSCodeのKeyBinding.jsonの設定
{
    "key": "cmd+l",
    "command": "workbench.action.nextEditor"
},
{
  "key": "cmd+h",
  "command": "workbench.action.previousEditor"
}
```

そのためvimで開いたバッファは `cmd` を押しながら`l` や `h` を押下することで、タブを切り替える用に移動できるように設定しております。

なお筆者は 左のoptキーをメタキーとして設定しています。その代わり全く使っていなかった右の cmdキーを左のoptionキーとして動作するように設定しております。

※ **以下gifでは `l` `h` としか表示されておりませんが、実際は `cmd`　を押しながら `l` や`h`を押下しています**

![vimのタブ移動](/images/nvim/vim_tab_movement.gif)

基本的な動作はこんな感じです。

`Ctrl+p` で表示されたかっこいい ファジーファインダーなどは別途プラグインの際にご紹介します。


## キーバインドの設定

### vimのキーバインド等

`cmd+l` でvimで開いた次のバッファに移動、 `cmd+h` でvimで開いた前のバッファに移動するためにvim側では以下のように設定しております。


```vim
" バッファの移動をメタキーで
nnoremap <silent> <M-h> :bprev<CR>
nnoremap <silent> <M-l> :bnext<CR>
nnoremap <silent> <M-w> :bd<CR>
```

これに加えて、以下に紹介するツールで、特定のアプリケーションを動かしている際にのみキーバインドを変更するようにしております。

### Karabiner-Elements

私はMacを利用しており、キーバインドの変更などはKarabiner-Elementsを利用しています。

https://karabiner-elements.pqrs.org/

以下のように、 `Iterm2` 、 `alacritty` , `wezterm` を利用している際に、以下のようにキーバインドを変更しております。

- `cmd` を押しながら `l` => `opt` を押しながら `l`
- `cmd` を押しながら `h` => `opt` を押しながら `h`

設定のjson(~/.config/karabiner/karabiner.json)は以下のように設定しております。

```json
{
  "description": "Command-l to Option-l on iTerm2",
  "manipulators": [
    {
      "type": "basic",
      "from": {
        "key_code": "l",
        "modifiers": {
          "mandatory": [
            "left_command"
          ]
        }
      },
      "to": [
        {
          "key_code": "l",
          "modifiers": [
            "left_option"
          ]
        }
      ],
      "conditions": [
        {
          "type": "frontmost_application_if",
          "bundle_identifiers": [
            "^com\\.googlecode\\.iterm2",
            "^io\\.alacritty",
            "^com\\.github\\.wez.wezterm"
          ]
        }
      ]
    }
  ]
}
// 同じような設定であるため hは省略
```

## vimで利用しているプラグインについて

私がvimを使うにあたり利用しているプラグイン・設定しているショートカットなどをご紹介します。

なお、ここではプラグイン管理の導入方法などはご紹介しておりませんのでご了承ください。

### プラグイン管理

dein.vim というプラグインを利用しています。

https://github.com/Shougo/dein.vim

dein自体の導入方法などは上記公式や別途記事をご参照ください。


### ファジーファインダー

telescope.vim というプラグインを利用しています。

VSCodeでいう `Cmd+p` でファイルを検索するような機能をイメージしてもらえれば良いと思います。

以下のようにファイル検索は `Ctrl+p`  をマッピングしております。

![コーディング風景](/images/nvim/file_finder.gif)

さらに `grep` のように特定の文字列が含まれたファイルなどを探している場合には、<Ctrl+g> で検索できるようにしております。

![コーディング風景](/images/nvim/grep_finder.gif)

このプラグインのすごいところは、ファイルのプレビューも表示してくれるという点です。(上記例では作業スペースが狭くて見えておりませんが・・・・)

以下のような形でプレビューをみることができます。

![コーディング風景](/images/nvim/fzf_preview.gif)

この点に関して、VSCodeの `Cmd+p` よりとても気に入っています。

またこの `ctrl + P` と `ctrl + G` とそれぞれ大文字で、プロジェクトルート(最初にvimで開いたディレクトリ)ではなく、現在バッファで開いているファイルのルートを起点に検索するためのキーバインドも入れております。

```vim
nnoremap <C-p> <cmd>Telescope find_files<cr>
nnoremap <C-g> <cmd>Telescope live_grep<cr>
" プロジェクトルートではなく現在開いているファイルを起点にファイル検索
nnoremap <C-P> <cmd>lua require('telescope.builtin').find_files( { cwd = vim.fn.expand('%:p:h') })<cr>
nnoremap <C-G> <cmd>lua require('telescope.builtin').live_grep( { cwd = vim.fn.expand('%:p:h') })<cr>

```

## ファイル横断での置換

VSCodeでいう以下画像のような一括置換の機能になります。

![VSCodeの置換](/images/nvim/vscode_replace.png)

これに関しては上記でご紹介した telescope.vim とvim-qfreplaceというプラグインを利用しています。

https://github.com/thinca/vim-qfreplace/blob/master/doc/qfreplace.jax

例えば以下のように `test` という文字列が含まれた複数のファイルについて、 `dest` という文字列に置き換える場合のオペレーションは以下のような形になります。

![全置換](/images/nvim/replace_all.gif)

結構手数が多いですが

- `telescope.vim` でファイル横断で `test` という文字列を検索
- `ctrl+a` で `test` という文字列が見つかった部分を vimのQuickfixに送る
- `vim-qfreplace` の機能で QuickFixに対して置換を行う
  - 上記では `test` という文字列を検索
  - `%s//dest/g` というコマンドで一括置換を行っています
  - `%s/test/dest/g` と同じことを行っています。

[参考]

vimgrep と QuickFixについて

https://qiita.com/yuku_t/items/0c1aff03949cb1b8fe6b

もちろんファイル単位で置換するか選択することもできますし、 1箇所ずつ置換するか確認できます。

今度は逆に `dest` という文字列を `test` に変換してみます

![選択置換](/images/nvim/replace_select.gif)

- `telescope.vim` でQuickFixに送るものを選択する場合 `tab` で選択してから、 `ctrl+q`で送ります
- 今度は `%s//dest/gc` として1箇所ずつ確認してから置換を行っています

以下のような設定をしています。

telescope

```vim
lua <<EOF
require('telescope').setup{
  defaults = {
    mappings = {
      i = {
# ctrl + a　で検出された結果全てをquickfixに送る
        ["<C-a>"] = require('telescope.actions').send_to_qflist + require('telescope.actions').open_qflist,
# ctrl + qで選択した部分をquickfixに送る
        ["<C-q>"] = require('telescope.actions').send_selected_to_qflist + require('telescope.actions').open_qflist
      }
    }
  }
}
EOF
```

qfreplace

```vim
nnoremap qf :Qfreplace<CR>
```


### ファイラー

`degx.nvim` というプラグインをファイラーとして利用しております。

https://github.com/Shougo/defx.nvim

- 新規でファイルを追加する時
- ファイル・ディレクトリをコピーする時
- ファイル・ディレクトリの名前を変更する時

などに利用しています。

`ctrl+n` でファイラーが開くようにしています。

![ファイラーの基本操作](/images/nvim/filer_basic.gif)

defxはかなり自由にカスタマイズすることができるようですが、私は以下のように設定を入れております。

- `N` で新規ファイルを作成
- `M` で新規ディレクトリを作成
- `c` でコピー
- `p` でペースト

defxの全設定はこちら。

https://github.com/bun913/dotfiles/blob/main/config/nvim/rc/defx.rc.vim

なお、VSCodeのサイドバーに表示されるファイラーでも似たような操作ができるようにキーバインドを以下のように設定しております。

```json
  // 新規ファイル作成
  {
    "key": "shift+n",
    "command": "explorer.newFile"
  },
  // 新規フォルダ作成
  {
    "key": "shift+m",
    "command": "explorer.newFolder"
  },
  // dでファイルを削除
  {
    "key": "d",
    "command": "deleteFile",
    "when": "explorerViewletVisible && filesExplorerFocus && !explorerResourceReadonly && !inputFocus"
  },
  // rでファイルのリネーム
  {
    "key": "r",
    "command": "renameFile",
    "when": "explorerViewletVisible && filesExplorerFocus && !explorerResourceIsRoot && !explorerResourceReadonly && !inputFocus"
  },
  {
    "key": "enter",
    "command": "-renameFile",
    "when": "explorerViewletVisible && filesExplorerFocus && !explorerResourceIsRoot && !explorerResourceReadonly && !inputFocus"
  },
  // vで分割して開く
  {
    "key": "v",
    "command": "explorer.openToSide",
    "when": "explorerViewletVisible && filesExplorerFocus && !explorerResourceIsRoot && !explorerResourceReadonly && !inputFocus"
  },
  // ファイルのコピー
  {
    "key": "y",
    "command": "filesExplorer.copy",
    "when": "explorerViewletVisible && filesExplorerFocus && !explorerResourceIsRoot && !explorerResourceReadonly && !inputFocus"
  },
  // ファイルのペースト
  {
    "key": "p",
    "command": "filesExplorer.paste",
    "when": "explorerViewletVisible && filesExplorerFocus && !explorerResourceIsRoot && !explorerResourceReadonly && !inputFocus"
  },
```

### git操作

私はターミナルで動作する `lazygit` を好んで使っています。
(Go製なのでどの環境でも動かしやすくてとても好き)

https://github.com/jesseduffield/lazygit

vimから `lazygit` を呼び出せるように 以下プラグインを利用しております。

https://github.com/kdheepak/lazygit.nvim

以下のように設定していて、 `<Space>gg` でlazygitが起動するようになっております。

```vim
nnoremap <silent> <leader>gg :LazyGit<CR>
```

![lazygit](/images/nvim/lazygit.gif)

### その他

今回は開発に当たっての操作感という趣旨ですので詳細は割愛いたしますが、以下のようなプラグインを入れており、もはや必須です。

LSP(言語ごとのインテリセンスやフォーマッターなど)

- vim-lsp
- vim-lsp-settings


mattnさんのプラグインは本当に神。

https://zenn.dev/mattn/articles/b83f9d94202914


言語ごとに色をつけたり、インデントを調整したりは以下のプラグインを利用しております。

- nvim-treesitter

https://github.com/nvim-treesitter/nvim-treesitter

https://zenn.dev/duglaser/articles/c02d6a937a48df

https://zenn.dev/monaqa/articles/2021-12-22-vim-nvim-treesitter-highlight


### 最後に

ライトvimmerですが、 `vimはいいぞ・・・`
