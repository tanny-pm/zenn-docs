# スクリーンショット撮影状況

Zenn本『Zed入門：AIと並走する次世代エディタのはじめかた』のスクリーンショット管理メモ。全23枚のうち**21枚撮影済み・2枚未撮影**。

- 画像の保存先：リポジトリ直下の `images/zed-book/<ファイル名>.png`
- 本文では `![alt](/images/zed-book/xxx.png)` の記法で参照
- 撮影済みの画像は本文の `📷 撮影メモ` コメントを削除済み。未撮影の画像には本文にコメントが残っている

## サマリー

| 章    | 撮影済み | 未撮影 | 計  |
| ----- | -------- | ------ | --- |
| 第1章 | 1        | 0      | 1   |
| 第2章 | 4        | 0      | 4   |
| 第3章 | 3        | 0      | 3   |
| 第4章 | 2        | 0      | 2   |
| 第5章 | 3        | 0      | 3   |
| 第6章 | 2        | 1      | 3   |
| 第7章 | 4        | 0      | 4   |
| 第8章 | 1        | 1      | 2   |
| 合計  | 21       | 2      | 23  |

## 第1章 why-zed.md

- [x] **ch1-overview**（必須）… プロジェクトパネル＋エディタ＋Agent Panel の全景

## 第2章 getting-started.md

- [x] **ch2-onboarding**（任意）… 初回起動のオンボーディング画面。ウェルカムページの「Return to Onboarding」から表示できた
- [x] **ch2-welcome**（必須）… ウェルカムページ。再表示は `zed: show welcome`
- [x] **ch2-command-palette-cli**（必須）… コマンドパレットに `cli` を入力し `cli: install cli binary` が候補に出た状態
- [x] **ch2-theme-selector**（任意）… テーマセレクタ（cmd-k cmd-t）で One Dark をプレビューした状態
- [x] **ch2-settings-editor**（必須）… GUI の設定エディタ（cmd-, で開く別ウィンドウ「Zed — Settings」。カテゴリ一覧・検索・トグル）

## 第3章 ai-agent.md

- [x] **ch3-agent-panel**（必須）… Agent Panel 全体（プロンプト入力済み）
- [x] **ch3-mention-menu**（必須）… 入力欄で `@` を打った補完メニュー
- [x] **ch3-tool-approval**（必須）… ツールカードの Allow / Deny / Only this time 承認 UI（Zed Pro トライアルで撮影）
- [x] **ch3-review-multibuffer**（任意）… 編集結果のレビュータブ（3ファイルのマルチバッファ差分）。`agent: open agent diff` で開いた

## 第4章 ai-editing.md

- [x] **ch4-inline-assistant**（必須）… ctrl-enter のプロンプト欄と diff 表示（Zed Pro トライアルで撮影）
- [x] **ch4-edit-prediction**（必須）… タイピング中にグレーで予測が出ている状態（Zeta はサインインのみで動作）

## 第5章 editor-core.md

- [x] **ch5-split-panes**（任意）… 左右2ペインに main.ts と formatter.ts を並べた状態
- [x] **ch5-file-finder**（必須）… ファイルファインダー（cmd-p）で `ufmt` を入力した状態
- [x] **ch5-search-multibuffer**（必須）… プロジェクト検索結果のマルチバッファ

## 第6章 language-tools.md

- [x] **ch6-diagnostics-panel**（必須）… Project Diagnostics パネル（型エラー3件）
- [x] **ch6-task-spawn**（必須）… task: spawn のタスク一覧モーダル
- [ ] **ch6-debugger**（任意）… ブレークポイントで停止中のデバッグセッション
  - **未撮影の理由**：デバッグ実行環境（対象スクリプト・アダプター）の準備が必要なため保留
  - **撮影方法**：スクリプトにブレークポイントを設定 → デバッガを起動して停止 → 変数パネルやコールスタックが見える状態を撮影

## 第7章 git-workflow.md

- [x] **ch7-git-panel**（必須）… Git Panel の全体像
- [x] **ch7-project-diff**（必須）… Project Diff ビュー（ctrl-g d）
- [x] **ch7-inline-blame**（任意）… inline blame がカーソル行に表示された状態
- [x] **ch7-conflict-ui**（任意）… コンフリクト解決 UI（Use HEAD / Incoming / Both ボタン）

## 第8章 customize.md

- [x] **ch8-extensions**（必須）… 拡張ギャラリー（cmd-shift-x）
- [ ] **ch8-mcp-indicator**（任意）… Agent Panel 設定ビューの MCP サーバー状態インジケーター
  - **未撮影の理由**：settings.json への MCP サーバー設定（context_servers）が必要なため保留
  - **撮影方法**：context_servers を settings.json に設定 → Agent Panel の設定ビューを開き、MCP サーバーのインジケーター（緑）が表示された状態を撮影

## 撮影のヒント（未撮影分を撮るとき）

- ウィンドウ単位のキャプチャは `cmd-shift-4` → `space` → ウィンドウをクリック
- 保存先は `images/zed-book/<ファイル名>.png`
- 撮影後、対応する本文の `<!-- 📷 撮影メモ ... -->` コメントを削除し、`npx prettier --write "books/zed-editor-intro-mac/*.md"` を実行
- ch3-tool-approval / ch3-review-multibuffer / ch4-inline-assistant は、Agent 用 LLM プロバイダー（Zed Pro トライアル or API キー）を設定すれば一度に撮れる
