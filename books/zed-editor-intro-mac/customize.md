---
title: "拡張機能とカスタマイズ"
---

本書もいよいよ最終章に差し掛かりました。ここまでの章で、Zedの中核を成す機能を一通り扱ってきました。AIエージェントによる開発、インラインアシスト、マルチバッファ、言語ツール、Gitとの連携です。この章では、それらをさらに自分好みに拡張し、手に馴染む道具へと仕上げるための手段を扱います。

Zedの拡張機能は、VS Codeのような巨大なマーケットプレイスとは少し趣が異なります。エディタの本体にすでに多くの機能が組み込まれているため、拡張で補う範囲は意図的に絞られています。言語サポート、テーマ、スニペット、デバッガー、そしてAI機能に接続するMCPサーバー。この5つが拡張の主な担い手です。設定ファイルとキーマップは、拡張とは別の軸を担う手段で、エディタの振る舞いをきめ細かく制御できます。

この章では、拡張のインストールと管理、MCPサーバーの接続、`settings.json` と `keymap.json` による設定とキーバインド、そしてスニペットまでを順に説明します。最後に、本書全体を振り返りながら「Zed流の開発スタイル」とはどういうものかを考えてみたいと思います。

## 拡張機能のエコシステム

### 拡張の種類

Zedの拡張機能は、現時点で以下の種類を提供できます。

- **Languages（言語）**：ツリーシッター文法やLanguage Server Protocolの統合により、新しいプログラミング言語のサポートを追加する。Rust、Go、TypeScript、Pythonなどは本体に組み込みで含まれており、拡張が必要になるのはニッチな言語や独自DSLが多い
- **Themes（テーマ）**：エディタの配色テーマ。ダークとライトを問わず、多数のテーマが公開されている
- **Icon Themes（アイコンテーマ）**：ファイルツリーに表示されるアイコンセットを切り替える
- **Snippets（スニペット）**：拡張としてスニペット集を配布できる。個人のスニペットは後述の設定ファイルで管理する
- **Debuggers（デバッガー）**：Debug Adapter Protocol（DAP）を実装したデバッガーを追加する（第6章参照）
- **MCP Servers（MCPサーバー）**：Model Context Protocolサーバーを拡張として提供し、Agent Panelに追加のツールを与える。次節で詳しく説明する

Zedの拡張は、Rust（WebAssemblyにコンパイル）で書かれた薄いラッパーと設定ファイルの組み合わせで実装されています。VS Codeの拡張が Node.js で任意のロジックを実行できるのに対し、Zed の拡張が担えることはこれらのカテゴリに限られています。拡張がエディタを遅くしないよう、Zedは意図的にその範囲を絞ったのです。

### 拡張のインストール

拡張のギャラリーを開くには、コマンドパレット（`cmd-shift-p`）で「`zed: extensions`」を実行するか、`cmd-shift-x` を押します。メニューから「Zed > Extensions」を選んでも構いません。

ギャラリーには公開されている拡張が一覧表示されます。検索ボックスで絞り込み、目当ての拡張の「Install」をクリックするだけでインストールが完了します。

インストールされた拡張は `~/Library/Application Support/Zed/extensions/` 以下に展開されます。ギャラリーの「Installed」タブには現在インストール済みの拡張が一覧され、そこから削除やアップデートも行えます。

![拡張ギャラリー。検索ボックスとカテゴリフィルタ、ダウンロード数付きの拡張一覧、Install ボタンが並ぶ](/images/zed-book/ch8-extensions.png)
_cmd-shift-x で開く拡張ギャラリー。インストールはワンクリックで完了する_

特定の拡張を自動インストールしたい場合は、`settings.json` に `auto_install_extensions` を設定することもできます。チームで共有する `.zed/settings.json` に記述しておけば、新メンバーが必要な言語拡張を自動で導入できます。

```jsonc
// settings.json（または .zed/settings.json）
{
  "auto_install_extensions": {
    "toml": true,
    "dockerfile": true
  }
}
```

## MCPサーバーとAI機能の拡張

第3章でAgent Panelを使ったAIエージェント開発を扱いましたが、そこではZed本体が提供するツール（ファイル操作、ターミナル実行など）を使いました。MCPサーバーを追加することで、そのツールセットを外部の任意のリソースに対して拡張できます。GitHubのIssue参照、データベースへの問い合わせ、社内Wikiへのアクセス。こうしたリソースをMCPサーバーとして接続すると、エージェントはそれらのツールを呼び出しながら作業を進められるようになります。

### settings.json でのMCPサーバー設定

MCPサーバーは `settings.json` の `context_servers` セクションに記述します。コマンドパレットで「`agent: add context server`」を実行するとモーダルが開き、設定の追加を案内してくれます。

ローカルで動作するMCPサーバー（CLIツールやnpmパッケージとして提供されるもの）は、次のように設定します。

```jsonc
{
  "context_servers": {
    "github-mcp-server": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_xxxxxxxxxxxx"
      }
    }
  }
}
```

HTTPやHTTPSで公開されているリモートのMCPサーバーには、URLで直接接続できます。

```jsonc
{
  "context_servers": {
    "my-remote-server": {
      "url": "https://example.com/mcp",
      "headers": {
        "Authorization": "Bearer <token>"
      }
    }
  }
}
```

設定後、Agent Panelの設定ビューを開くとサーバーの状態がインジケーターで表示されます。緑であれば正常に動作しています。

![Agent Panel 設定ビューのMCPサーバー状態インジケーター。サーバー名の横に緑の丸が表示されている](/images/zed-book/ch8-mcp-indicator.png)
_Agent Panel 設定ビューでMCPサーバーの接続状態を確認できる。緑が正常動作の印_

<!-- 📷 撮影メモ【任意】: context_servers を settings.json に設定した後、Agent Panel の設定ビューを開いてMCPサーバーのインジケーターが表示された状態を撮影。撮影後 images/zed-book/ch8-mcp-indicator.png に保存し、このコメントを削除 -->

### 拡張としてのMCPサーバーについて

一部のMCPサーバーは拡張ギャラリーから直接インストールできる形式で公開されていました。ただし、2025年後半以降、ZedはMCPサーバー拡張を公式MCPレジストリに移行する方針を示しており、今後は `context_servers` への直接記述か、公式レジストリからの追加が主な方法になる見込みです。最新の動向は公式ドキュメントを確認してください。

## settings.json の深掘り

### 設定ファイルの場所と優先順位

Zedの設定は3つの層で管理されます。

1. **グローバル設定**：`~/.config/zed/settings.json`。コマンドパレットで「`zed: open settings`」を実行すると開く。全プロジェクトに共通の設定をここに書く
2. **言語別設定**：グローバル設定の `languages` キーの下に、言語名をキーとして記述する。特定言語だけに適用される
3. **プロジェクト別設定**：プロジェクトルートの `.zed/settings.json`。このファイルがあると、そのプロジェクトを開いたときにのみ適用され、グローバル設定を上書きする

優先順位は、プロジェクト別 > 言語別 > グローバルの順です。チームのコーディング規約（インデント幅など）をプロジェクト別設定に書いておくと、参加メンバー全員が同じ設定でエディタを動かせます。

### よく使う設定の実例

以下に、実際の開発でよく調整する設定項目を示します。

```jsonc
// ~/.config/zed/settings.json
{
  // フォント
  "buffer_font_family": "Zed Plex Mono",
  "buffer_font_size": 14,

  // エディタ動作
  "autosave": "on_focus_change",
  "format_on_save": "on",
  "tab_size": 2,
  "hard_tabs": false,

  // UI（テーマはオブジェクト形式で指定する）
  "theme": {
    "mode": "dark",
    "dark": "One Dark",
    "light": "One Light"
  },
  "icon_theme": {
    "mode": "system",
    "dark": "Zed (Default)",
    "light": "Zed (Default)"
  },

  // ターミナル
  "terminal": {
    "font_size": 13,
    "shell": {
      "program": "/bin/zsh"
    }
  }
}
```

### 言語別設定

`languages` セクションでは、言語名（大文字小文字に注意）をキーに、上書きしたい設定値を記述します。

```jsonc
{
  "languages": {
    "Python": {
      "tab_size": 4,
      "format_on_save": "on",
      "formatter": "language_server"
    },
    "Go": {
      "tab_size": 4,
      "hard_tabs": true,
      "format_on_save": "on"
    },
    "Markdown": {
      "soft_wrap": "editor_width",
      "format_on_save": "off"
    }
  }
}
```

Pythonのインデントは4スペース、GoはTABで統一、Markdownは自動折り返しを有効にして保存時フォーマットは無効、といった設定をそれぞれ使い分けられます。

### プロジェクト別設定

チームで運用するリポジトリでは、`.zed/settings.json` をコミットしておくことを検討したいところです。以下はフロントエンドプロジェクトの例です。

```jsonc
// .zed/settings.json（プロジェクトルートに配置）
{
  "tab_size": 2,
  "format_on_save": "on",
  "auto_install_extensions": {
    "typescript": true,
    "tailwindcss": true
  },
  "languages": {
    "TypeScript": {
      "formatter": {
        "external": {
          "command": "prettier",
          "arguments": ["--stdin-filepath", "{buffer_path}"]
        }
      }
    }
  }
}
```

この設定ファイルをリポジトリに含めておけば、チームメンバーが同じZedの設定でプロジェクトを開発できます。「自分のマシンでは動いたのに」を減らす、地味ですが効果的な手段です。

## keymap.json によるキーバインドカスタマイズ

### context の概念

Zedのキーバインドには `context`（コンテキスト）という仕組みがあります。同じキーを押しても、現在フォーカスが当たっている場所に応じて異なるアクションが実行されるよう、条件を記述できます。

コンテキストはZedのUI要素の階層に対応しています。ルートはWorkspace、その下にPaneやPanel、さらにEditorやTerminalなどが続きます。`context` 欄に文字列を書くと、その要素にフォーカスがあるときだけキーバインドが有効になります。

コンテキスト式では次の演算子が使えます。

- `&&`（AND）、`||`（OR）、`!`（NOT）
- `mode == full` のような等値比較
- `>` による親子関係の指定

例えば、「エディタでも、ターミナルでもないとき」は `!Editor && !Terminal` と書きます。「フルスクリーンモードのエディタ」なら `Editor && mode == full` です。

### keymap.json を編集する

コマンドパレットで「`zed: open keymap`」を実行すると `~/.config/zed/keymap.json` が開きます。JSONの配列として記述し、各エントリに `context`（省略可）と `bindings` を指定します。

```jsonc
// ~/.config/zed/keymap.json
[
  {
    // コンテキストなし：どこでも有効
    "bindings": {
      "cmd-shift-e": "project_panel::ToggleFocus"
    }
  },
  {
    // エディタにフォーカスがあるとき
    "context": "Editor",
    "bindings": {
      "cmd-d": "editor::SelectNext",
      "ctrl-shift-k": "editor::DeleteLine"
    }
  },
  {
    // プロジェクトパネルが編集状態でないとき
    "context": "ProjectPanel && not_editing",
    "bindings": {
      "a": "project_panel::NewFile",
      "shift-a": "project_panel::NewDirectory",
      "r": "project_panel::Rename",
      "d": "project_panel::Delete"
    }
  },
  {
    // ターミナルにフォーカスがあるとき
    "context": "Terminal",
    "bindings": {
      "cmd-k": "terminal::Clear"
    }
  }
]
```

既存のキーバインドを無効にしたい場合は、`bindings` の値に `null` を指定します。

```jsonc
[
  {
    "context": "Editor",
    "bindings": {
      "cmd-k cmd-d": null
    }
  }
]
```

利用可能なアクション名は、コマンドパレットの一覧や公式ドキュメントで確認できます。また、`zed: open default keymap` を実行するとZedのデフォルトキーマップが表示され、どんな設定がデフォルトで入っているかを参照できます。

## スニペット

スニペットは、定型のコードブロックをトリガーワードから素早く展開するための機能です。`snippet: configure snippets` をコマンドパレットで実行すると、`~/.config/zed/snippets/` ディレクトリ内のスニペットファイルを作成・編集できます。

スニペットファイルはJSON形式で、ファイル名によってスコープ（適用される言語）が決まります。`snippets.json` はすべての言語に、`typescript.json` はTypeScriptにのみ適用されます。

```jsonc
// ~/.config/zed/snippets/typescript.json
{
  "Arrow function": {
    "prefix": "af",
    "body": ["const ${1:name} = (${2:args}) => {", "  ${0}", "};"],
    "description": "アロー関数の雛形"
  },
  "React functional component": {
    "prefix": "rfc",
    "body": [
      "type ${1:Props} = {",
      "  ${2}",
      "};",
      "",
      "export function ${3:Component}({ ${4} }: ${1:Props}) {",
      "  return (",
      "    <div>",
      "      ${0}",
      "    </div>",
      "  );",
      "}"
    ],
    "description": "Reactの関数コンポーネント"
  }
}
```

`$1`、`$2` などはタブストップで、`tab` キーで次へ移動できます。`$0` が最終的なカーソル位置です。`${1:name}` のように書くと、そのタブストップにデフォルト値のプレースホルダーが入ります。

スニペット拡張をインストールすると、コミュニティが整備したスニペット集も使えるようになります。頻繁に書くコードのパターンは自分のスニペットとして溜めていくと、作業のリズムが上がります。

## Zed流の開発スタイル

本書を読み始めたとき、Zedはどんな印象だったでしょうか。「また新しいエディタか」「Rustで速いのはわかったが、乗り換えるほどのものか」。正直そう思っていたかもしれません。

第1章でZedの背景を追い、第2章でMacへのセットアップを済ませ、第3章のAgent PanelでAIエージェントが動き出したとき、何かが変わったのではないかと思います。コードを書くのではなく、エージェントに意図を伝えて成果物を受け取る。その感覚は、これまでのエディタの使い方とは質的に異なります。第4章のインラインアシストとEdit Predictionは、その体験をより小粒な単位にまで浸透させてくれます。

エディタ機能の深掘り（第5章）と言語ツール（第6章）、Gitとの連携（第7章）を経て、本章でカスタマイズの手段を学んだいま、Zedはインストールした直後の見知らぬ道具から、自分の手に馴染んだ道具になっているはずです。

Zedでの開発を一言で表すなら、「人間とAIが同じバッファの上で並走する」だと思います。エージェントはファイルを開き、コードを書き、テストを走らせ、エラーを読んで修正します。人間はその過程に立ち会い、方向を決め、判断を下します。ターミナルとエディタとチャットが別ウィンドウに散らばらず、一つの画面の中で自然に流れていく。Zedはそういうツールを作ろうとしています。

カスタマイズで整えた設定ファイルは、その作業の土台です。フォントの選択も、保存時の自動フォーマットも、キーバインドも、スニペットも、それぞれは小さな決定ですが、毎日使うエディタの体験を少しずつ良くしていきます。自分が快適に感じる環境は、長い目で見れば確実に効いてきます。

乗り換えに迷っている人には、まず1週間、メインのプロジェクトをZedで開いてみることをすすめます。Agent Panelに慣れると、おそらく他のエディタで開発する気がしなくなります。Zedはこれからも速くなり、機能を増やし、洗練されていくでしょう。本書がその出発点になれば、うれしく思います。
