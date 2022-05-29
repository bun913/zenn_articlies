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

ただ VSCodeを使う時にもnvimを使う時にも似たような開発体験をするためには、各種プラグインの導入・キーバインドの設定などが必要であるため、1週間くらいかけて諸々抜本的なnvim環境の作成し直しを行いました。

同僚「**そんなことするくらいならVSCodeでいいじゃん・・・**」

僕「**こんなにかっこよく育ってくれたターミナルとnvimの前でよくそんなこと言えるな！！この子の前でもう一度言ってみろ！！**」

![コーディング風景](/images/nvim/terminal.jpeg)

同僚(クソデカため息)「**そんなことするくらいならVSCodeでいいじゃん・・・**」

というやりとりもあったとか。

※ **今回ご紹介している各種設定はそもそも vim , VSCodeを同じように利用しようとしなければ良いだけの話です。「変なことしている奴がいるなぁ・・・と生優しく見守っていただければ幸いです」**

## コーディングの基本道具・基本動作

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

vimで開いたバッファは `cmd` を押しながら、さながらタブを切り替える用に移動しています。

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

```

## vimで利用しているプラグインについて
