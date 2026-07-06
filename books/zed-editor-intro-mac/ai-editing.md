---
title: "インラインアシストとEdit Prediction"
---

第3章では、Agent Panelを通じてZedのAIと対話しながらコードを進める方法を見てきました。Zedにはそれとは別に、編集の流れに溶け込む形のAI機能がもう2つあります。インラインアシスタントとEdit Predictionです。

インラインアシスタントは、「いまここを書き換えてほしい」という単発の指示をエディタ上で完結させます。チャットウィンドウを開く必要はなく、カーソルの置かれたコードをそのままAIへ渡して、その場で置換してもらえます。Edit Predictionはさらに静かで、タイピングに合わせて次に書くべきコードを薄くグレーで示し続けます。プロンプトを書く必要もなく、Tabキーを押すだけで提案を受け入れられます。

どちらも小さな機能に見えますが、使い始めると開発のリズムがじわじわ変わります。この章ではこの2つの機能に加え、AIへの継続的な指示を置くInstructionsファイル、プライバシーに関わるデータの扱い、settings.jsonの関連設定についても扱います。

## インラインアシスタント

### 起動と基本操作

インラインアシスタントを起動するキーバインドは `ctrl-enter` です。エディタのどこでも、テキストを選択しているかどうかにかかわらず、このキーを押すとプロンプト入力欄がバッファの中に展開されます。

選択範囲がある場合、その範囲がAIに渡すコンテキストになり、生成結果で置換されます。選択がない場合はカーソル行が対象になります。たとえば関数の中にカーソルを置いて `ctrl-enter` を押し、「この関数にエラーハンドリングを追加して」と入力してEnterを押すと、関数全体がAIによって書き換えられます。

生成された変更はdiff形式でその場に表示され、受け入れるか破棄するかを選べます。受け入れは `escape` で確定、破棄は `ctrl-z` で元に戻ります（詳細な操作はZedのキーマップ設定で変更可能です）。

![インラインアシスタントのプロンプト欄とdiff表示。バッファ内にプロンプト欄が展開され、変更箇所がdiff形式で示される](/images/zed-book/ch4-inline-assistant.png)
_ctrl-enter で起動するインラインアシスタント。プロンプト欄がバッファ内に展開され、生成結果がそのままdiffで表示される_

インラインアシスタントで利用するモデルは、settings.jsonの `agent.inline_assistant_model` で個別に指定できます。Agent Panelとは別のモデルを割り当てたい場面で役立ちます。

### コンテキストの追加

プロンプトの中では `@` メンションを使って追加のコンテキストを渡せます。`@ファイル名`、`@ディレクトリ名`、`@シンボル名`のほか、`@thread` で過去のAgent Panelスレッドを参照することもできます。

たとえば「`@types/user.ts` の型定義に合わせてこのオブジェクトを修正して」のように書くと、別ファイルの内容を参照しながらその場の書き換えを行えます。慣れると、複雑な変更はAgent Panelへ、局所的な変換はインラインアシスタントへ、と自分の中で使い分けが決まってきます。

### ターミナルパネルでの利用

インラインアシスタントはターミナルパネルでも同じ `ctrl-enter` で起動します。エラーメッセージを選択して `ctrl-enter` を押し、「このエラーの原因を調べるコマンドを書いて」と頼む。エディタでやることと変わりません。エディタとターミナルで操作が統一されているので、コンテキストスイッチなしに使えます。

### 複数カーソルとモデルの並列生成

Zedのマルチカーソル機能（第5章で詳しく扱います）と組み合わせると、複数の場所へ同じプロンプトを同時に適用できます。複数のカーソルを置いた状態で `ctrl-enter` を押すと、すべてのカーソル位置に対して並列で生成が走り、それぞれの文脈に合わせた変更が提示されます。

さらに、settings.jsonに `agent.inline_alternatives` を設定すると、異なるモデルから同時に候補を生成して比較できます。

```jsonc
// settings.json
{
  "agent": {
    "inline_alternatives": [{ "provider": "zed.dev", "model": "gpt-5-mini" }]
  }
}
```

### プリセットプロンプト

よく使う指示はキーバインドとして登録しておけます。keymap.jsonに以下のような設定を追加すると、特定のキー操作で決まったプロンプトが入力済みの状態でインラインアシスタントが開きます。

```jsonc
// keymap.json
[
  {
    "bindings": {
      "ctrl-shift-enter": [
        "assistant::InlineAssist",
        { "prompt": "このコードにドキュメントコメントを追加して" }
      ]
    }
  }
]
```

ドキュメントコメントの付与、テストの雛形生成、リファクタリングといった定型作業をワンキーで呼び出せるようになります。

## Edit Prediction

### Zetaモデルによる予測補完

Edit PredictionはZedが独自に開発したオープンソースモデル「Zeta」を使い、タイピングしながら次のコードを予測する機能です。コード補完ツールに近い感覚で使えますが、単語単位ではなく、より広い範囲の変更を予測して提案する点が特徴です。

Zetaモデルを使うにはZedアカウントへのサインインが必要です。サインインするだけで予測が自動的に有効になり、タイピング中にグレーのテキストとして提案が表示されはじめます。

![関数を書き始めると、次の行がグレーの予測テキストで表示され、tab Accept のヒントが添えられている](/images/zed-book/ch4-edit-prediction.png)
_Edit Prediction の予測表示。関数のシグネチャを書いた時点で中身が提案され、tab で受け入れる_

### プランと利用回数

Edit Predictionの利用回数はZedのプランによって異なります。

- **無料プラン**：月2,000回まで（Zeta予測）
- **Proプラン**：無制限

詳細な料金は [zed.dev/pricing](https://zed.dev/pricing) で確認できます。毎月2,000回という上限はアクティブな開発者であればさほど多い数ではありません。頻繁に使うようであれば早めにProプランへの移行を検討するとよいでしょう。

### 予測の受け入れ方

表示された予測を受け入れるには `tab` を押します（補完メニューが開いていない場合）。`alt-tab` も同じく受け入れに使えます。

提案全体ではなく一部だけを受け入れたい場合には、次の操作が使えます。

- **次の単語まで受け入れ**：`ctrl-cmd-right` または `alt-k`
- **行末まで受け入れ**：`ctrl-cmd-down` または `alt-j`
- **予測を手動で表示**（表示されていない場合）：`alt-\`

予測を受け入れず無視したい場合は、そのまま別のキーを打ち続けるだけです。自分の入力と予測が食い違っていると感じたときも、タイプし続ければ予測は更新されます。

### 表示モードの調整

Edit Predictionの表示には2つのモードがあります。

- **eager**（デフォルト）：予測が自動的にインライン表示される。言語サーバーの補完メニューと競合するときは表示を控える。
- **subtle**：修飾キー（デフォルトはAlt）を押している間だけ予測が現れる。予測の表示が気になる場合に向く。

settings.jsonで切り替えられます。

```jsonc
// settings.json
{
  "edit_predictions": {
    "mode": "subtle"
  }
}
```

### 代替プロバイダー

Zetaモデル以外のプロバイダーも選べます。GitHub Copilot、Mercury Coder（Inception Labs）、Codestral（Mistral）に加え、OllamaなどOpenAI互換のローカルモデルにも対応しています。

```jsonc
// settings.json（GitHub Copilotを使う場合）
{
  "edit_predictions": {
    "provider": "copilot"
  }
}
```

ローカルモデルを使う場合は以下のように詳細を設定します。

```jsonc
// settings.json（ローカルモデルの例）
{
  "edit_predictions": {
    "provider": "ollama",
    "ollama": {
      "api_url": "http://localhost:11434",
      "model": "qwen2.5-coder:7b-base",
      "prompt_format": "infer",
      "max_output_tokens": 512
    }
  }
}
```

## プライバシーとデータの扱い

### デフォルトの動作

Zedはデフォルトではプロンプトもコードのコンテキストも保持しません。Zedホスト型モデルを利用している場合、各プロバイダーとの契約によりゼロデータ保持が義務付けられています（一部の安全審査用モデルを除く）。プロバイダーのAPIキーを直接使う場合は、そのプロバイダーのプライバシーポリシーが適用されます。APIキー自体はmacOSのシステムキーチェーンに保存されます。

Edit Predictionは入力のたびにコードをモデルへ送信しますが、通常はそこで処理が終わり、データは保持されません。

### トレーニングデータへのオプトイン

Edit Predictionの改善に貢献する形で自分のコードをトレーニングデータとして提供する仕組みがありますが、これはオプトイン制です。以下の3条件をすべて満たした場合のみデータが収集されます。

1. ステータスバーのEdit Predictionメニューから「Training Data Collection」を手動で有効化している
2. プロジェクトにライセンスファイルが存在する（オープンソースプロジェクトの確認）
3. ファイルが `edit_predictions.disabled_globs` で除外されていない

オプトインしている場合でも、`.env*`、`*.pem`、`*.key`、`*.cert`、`secrets.yml` などの機密性の高いファイルは常に収集対象から除外されます。

特定のファイルやディレクトリをEdit Predictionそのものから除外したい場合（トレーニングへの提供ではなく、予測自体を行わせたくない場合）は `disabled_globs` を設定します。

```jsonc
// settings.json
{
  "edit_predictions": {
    "disabled_globs": ["~/.config/zed/settings.json", "**/.env*"]
  }
}
```

## Instructionsファイル

### RulesからInstructionsへ

以前のZedには `.rules` ファイルでAIへの指示を書く「Rules」機能がありました。現在このシステムはInstructionsとSkillsに置き換えられています。Instructionsは、エージェントとのすべてのやり取りに常時適用されるコンテキストです。

### ファイルの種類と場所

Instructionsには個人レベルとプロジェクトレベルの2種類があります。

**個人設定**（すべてのプロジェクトに適用）：

```
~/.config/zed/AGENTS.md
```

**プロジェクト設定**（プロジェクトルートに配置）：

Zedは以下のファイルを優先順に探し、最初に見つかったものを使います。

1. `.rules`
2. `.cursorrules`
3. `.windsurfrules`
4. `.clinerules`
5. `.github/copilot-instructions.md`
6. `AGENT.md`
7. `AGENTS.md`
8. `CLAUDE.md`
9. `GEMINI.md`

プロジェクト設定は個人設定より優先されます。他のAIツールで `CLAUDE.md` や `.cursorrules` を使っているプロジェクトであれば、そのファイルがZedでもそのまま機能します。

### Instructionsの書き方

ファイルはMarkdown形式で書きます。「このプロジェクトではTypeScriptを使う」「エラーハンドリングは必ずResult型で」「コメントは日本語で」といった、プロジェクト固有のルールや好みを書いておきます。

```md
# プロジェクト指示

- 言語：TypeScript（strict mode）
- テストフレームワーク：Vitest
- コメントは日本語で書く
- 関数にはJSDocを付ける
- 非同期処理はasync/awaitを使い、Promiseチェーンは避ける
```

このファイルはエージェントとのすべての会話に自動で組み込まれます。毎回「このプロジェクトの規約は…」と説明する手間が省けますし、AIが好き勝手に別の書き方を提案してくる頻度も減ります。

## settings.jsonでのAI設定

ZedのAI関連設定の多くはsettings.jsonで管理します。主要なキーをまとめます。

```jsonc
// settings.json（AI関連の主要設定）
{
  // エージェント関連
  "agent": {
    // インラインアシスタント用モデル（省略時はAgentのデフォルトモデルを使用）
    "inline_assistant_model": {
      "provider": "zed.dev",
      "model": "claude-opus-4-5"
    },
    // コミットメッセージ生成用モデル
    "commit_message_model": {
      "provider": "zed.dev",
      "model": "claude-haiku-4-5"
    },
    // インラインアシスタントの並列生成モデル
    "inline_alternatives": [],
    // スレッドの自動圧縮
    "auto_compact": true
  },

  // Edit Prediction関連
  "edit_predictions": {
    // プロバイダー："zed"（Zeta）, "copilot", "none"
    "provider": "zed",
    // 表示モード："eager" または "subtle"
    "mode": "eager",
    // 予測を無効化するファイルパターン
    "disabled_globs": []
  }
}
```

言語ごとにEdit Predictionを無効化する場合は言語設定に記述します。

```jsonc
// settings.json
{
  "languages": {
    "Markdown": {
      "show_edit_predictions": false
    }
  }
}
```

## AI機能を無効化・制限したい場合

特定の場面でAIをオフにしたい、あるいはプロジェクト全体でAIを使わないようにしたいケースもあるでしょう。

**Edit Predictionを完全に無効化する**：

```jsonc
{
  "edit_predictions": {
    "provider": "none"
  }
}
```

**特定言語だけ無効化する**（MarkdownやYAMLなど、予測がかえって邪魔な場合）：

```jsonc
{
  "languages": {
    "Markdown": { "show_edit_predictions": false },
    "YAML": { "show_edit_predictions": false }
  }
}
```

**機密ファイルをEdit Predictionから除外する**：

```jsonc
{
  "edit_predictions": {
    "disabled_globs": ["**/.env*", "**/secrets/**", "~/.config/**"]
  }
}
```

インラインアシスタントはモデルプロバイダーの設定を削除すれば無効化できますが、Edit Predictionほど細かく制御したいケースは実際には少ないと思います。セキュリティポリシーが厳しいプロジェクトで作業するときは、まず `disabled_globs` で機密ファイルを除外しておくだけで十分なことがほとんどです。

---

次の第5章では、マルチバッファ編集や検索・ナビゲーションなど、エディタ機能の深掘りに入ります。
