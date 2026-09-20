# AperCode

[English](README.en.md)

ローカル LLM（Ollama）で動く汎用コーディングエージェント。Web ブラウザ UI 付き。
MT5 (wine) の EA コンパイル・バックテストをツールとして組み込み済み。
コンテキスト自動圧縮とサブエージェント（子エージェント）に対応し、中規模プロジェクトの構築を想定。

## 構成

```
codeagent/
├── app.py                  # デスクトップ起動（サーバー自動起動 + 専用ウィンドウ）
├── install_desktop.sh      # アプリメニュー / デスクトップへの登録
├── main.py                 # サーバーのみ起動
├── config.example.yaml     # 設定テンプレート（config.yaml にコピーして編集）
├── LICENSE                 # MIT
├── config.yaml             # 設定（モデル・権限・MT5 パス）
├── codeagent/
│   ├── agent.py            # エージェントループ（LLM ⇄ ツール）
│   ├── llm.py              # Ollama / Anthropic クライアント
│   ├── prompts.py          # システムプロンプト
│   ├── session.py          # 会話履歴の保存（JSON）
│   ├── memory/store.py     # 経験メモリ（SQLite FTS5 + 埋め込みベクトル）
│   ├── tools/
│   │   ├── fs.py           # read_file / write_file / edit_file / list_dir / glob / grep
│   │   ├── shell.py        # run_command
│   │   ├── mt5.py          # mt5_compile / mt5_backtest
│   │   ├── subagent.py     # spawn_agent（子エージェント）
│   │   └── memory_tools.py # search_memory / save_memory
│   └── server/
│       ├── app.py          # FastAPI + WebSocket
│       └── static/index.html
```

## セットアップ

```bash
cd codeagent
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp config.example.yaml config.yaml     # 任意。無ければ config.example.yaml がそのまま使われる
ollama pull qwen3:8b bge-m3            # 使うモデルと埋め込みモデル
```

設定は 3 段階で読み込まれる（後のものが優先）:
1. `config.example.yaml`（リポジトリ同梱の既定値）
2. `config.yaml`（プロジェクト直下。`.gitignore` 済み）
3. `~/.apercode/config.yaml`（ユーザー固有の上書き。差分だけ書けばよい）

`config.yaml` の主な項目:

| 項目 | 内容 |
|---|---|
| `llm.model` | 使うモデル（例: `gemma4:26b`） |
| `llm.tool_mode` | `native`（Ollama tool calling）/ `json`（非対応モデル用フォールバック） |
| `workspace.root` | エージェントが触れる範囲。この外は拒否される |
| `workspace.default_project` | 新規セッションの既定プロジェクト。空なら最近使ったものを使う |
| `permissions.*` | ツールごとに `allow` / `ask` / `deny` |
| `mt5.*` | WINEPREFIX と MT5 のインストール先 |
| `compaction.*` | コンテキスト圧縮の閾値（num_ctx に対する割合）と、残す直近メッセージ数 |
| `subagents.*` | 子エージェントの上限回数・深さ・モード別モデル |
| `memory.*` | 経験メモリ: 埋め込みモデル、自動注入件数、自動抽出の条件 |

## 起動

### デスクトップアプリとして（推奨）

```bash
bash install_desktop.sh
```

アプリメニュー（開発）とデスクトップに「AperCode」が追加される。クリックすると
サーバーを自動起動し（Ollama が止まっていれば `ollama serve` も試みる）、専用ウィンドウを開く。
ウィンドウを閉じるとサーバーも終了する。ログは `~/.apercode/app.log`。

専用ウィンドウの優先順位:
1. pywebview GTK バックエンド（OS の WebKit2GTK。日本語入力がそのまま使える。`bash setup_linux_window.sh` で準備）
2. Chromium / Chrome / Brave の `--app` モード（アドレスバー無しのウィンドウ）
3. pywebview Qt バックエンド（`pip install "pywebview[qt]"`。pip 版 Qt は fcitx 未対応のため日本語入力に難あり）
4. 既定のブラウザ

Linux で日本語入力（Mozc 等）を専用ウィンドウで使うには、1 を推奨:

```bash
bash setup_linux_window.sh   # python3-gi / WebKit2GTK を入れ、.venv を --system-site-packages で作り直す
```

ターミナルから同じ動作をさせるには `python3 app.py`（`--browser` で常に既定ブラウザ）。

#### トラブルシューティング

**専用ウィンドウで日本語入力（Mozc など）が起動しない**

pip でインストールした Qt（`pywebview[qt]`）には fcitx 用の入力メソッドプラグインが含まれていないため、
Qt バックエンドの専用ウィンドウ内では IME が動かない。ブラウザ（Firefox 等）では問題なく入力できるのが特徴。

対策: OS 標準の GTK + WebKit2GTK を使う GTK バックエンドに切り替える。

```bash
bash setup_linux_window.sh
```

このスクリプトは `python3-gi` と WebKit2GTK のバインディングを apt で入れ、仮想環境を
`--system-site-packages` 付きで作り直し（依存パッケージも再インストール）、Qt を外す。
以後 `app.py` は GTK バックエンドを最優先で使うので、OS の日本語入力がそのまま効く。

**専用ウィンドウが開かず、ブラウザで開いてしまう**

`~/.apercode/app.log` に理由が記録される。Qt バックエンドで
`Could not load the Qt platform plugin "xcb"` と出る場合は `sudo apt install -y libxcb-cursor0`。

**変換確定の Enter で送信されてしまう**

修正済み（IME 変換中の Enter は無視する）。古い `index.html` を使っている場合は最新版に更新すること。

## サーバーだけ起動する（開発用）

```bash
python3 main.py
# → http://127.0.0.1:8765 をブラウザで開く
```

## 使い方

1. 左上でモデルを選び、作業ディレクトリを指定して「新しいセッション」
   - 📁 ボタンでディレクトリ選択ダイアログ（最近使ったもの・新しいフォルダの作成も可）
   - 存在しないパスを入力すると「新規に作成しますか？」と確認される
   - 選べるのは `workspace.root` 配下のみ
2. 指示を入力（例: 「USDJPY H1 の移動平均クロス EA を作ってコンパイルして」）
3. `write_file` / `edit_file` / `run_command` は承認ダイアログが出る。「このセッションでは常に許可」で以降スキップ可能
4. ツールカードをクリックすると引数と結果が展開される
5. 「停止」で処理を中断

画像やテキストの添付:
- 画像は入力欄に Ctrl+V で貼り付け、またはファイルをドラッグ&ドロップ、📎 ボタンから選択
- 3,000 文字を超えるテキストを貼り付けると、本文ではなく添付（📄）として扱われる
- 画像はセッションごとに `~/.apercode/sessions/<id>_files/` に保存され、直近 2 回分のみ LLM に渡す
- 画像を理解できるのは vision 対応モデル（例: `qwen3-vl`, `gemma3`, `llava`）のみ。非対応モデルでは画像を除いて送信し、その旨を表示する

プロジェクト直下に `AGENT.md`（または `CLAUDE.md`）を置くと、その内容がシステムプロンプトに読み込まれる。EA の命名規則や共通ライブラリの説明などを書いておくと精度が上がる。

## コンテキスト圧縮

履歴の推定トークン数（または直近の実測値）が `num_ctx × compaction.threshold` を超えると、
直近 `keep_recent` 件を残して古い履歴を LLM で要約し、1 メッセージに置き換える。
元のメッセージはセッション JSON の `archive` に保存される。
ヘッダー右上のメーターで現在の使用量を確認でき、「要約」ボタンで手動実行もできる。

## サブエージェント

親エージェントは `spawn_agent` ツールで子エージェントにサブタスクを委譲できる。
子は独立した会話履歴で動き、完了時に「結果報告」だけを親に返すので、親のコンテキストを消費しない。

- `mode=explore`: 読み取り専用（read_file / list_dir / glob / grep）。既存コードの調査向け
- `mode=code`: 編集・コマンド実行可。独立したモジュールの実装向け

承認が必要な操作は子でも同じように UI に承認ダイアログが出る。
`subagents.explore_model` / `code_model` で子だけ別モデル（軽量モデルなど）を使える。

## 経験メモリ

成功・失敗の教訓や環境固有の事実を蓄積し、次のタスクで自動的に参照する。分野（言語・ツール）に依存しない。

- 保存単位: `状況 / 行動 / 結果(成功・失敗・事実) / 教訓 / タグ / プロジェクト / 確認済みフラグ`
- 蓄積の経路
  - 自動: ツールを `reflect_min_tools` 回以上使ったターンの終了後、LLM が作業記録から教訓を抽出して保存（`memory.reflect: auto`）
  - エージェント自身: `save_memory` ツール
  - 手動: 左メニュー「経験メモリ」から追加・編集・削除・確認済み設定
- 参照の経路
  - 自動: ユーザー入力ごとに関連する経験を検索し、上位 `inject_top_k` 件をシステムプロンプトに注入（画面に「参照した経験」と表示）
  - エージェント自身: `search_memory` ツール
- 検索: 埋め込みベクトル（Ollama `bge-m3`）と全文検索（FTS5 trigram）のハイブリッド。プロジェクトの言語タグ一致・確認済みにボーナス
- 類似度 `dedup_threshold` 以上の既存メモリがあれば重複として保存しない

埋め込みモデルの準備:

```bash
ollama pull bge-m3
```

モデルが無い場合は全文検索のみで動作する（「経験メモリ」画面に「利用不可」と表示される）。
`embed_model` を変えたら「再索引」ボタンで埋め込みを再計算する。

自動抽出された教訓は `verified=0`（未確認）で保存される。管理画面で「確認済にする」を押すと検索時に優先される。
誤った教訓は削除すること。DB は `~/.apercode/memory.sqlite`（1 ファイル、バックアップはコピーするだけ）。

## 中規模プロジェクトを作るときのコツ

1. 最初に要件と設計を相談し、エージェントに `AGENT.md`（設計方針・構成・進捗）を書かせる
2. 機能単位で指示を出す。「全部作って」より「まず○○モジュールを作ってテストして」
3. 大きな調査や独立モジュールは子エージェントに任せるよう促す
4. セッションが長くなったら新しいセッションを作る。`AGENT.md` があれば引き継げる

## tool calling が使えないモデルの場合

`llm.tool_mode: json` にすると、モデルに次の形式でツール呼び出しを書かせて解析する:

````
```tool_call
{"name": "read_file", "arguments": {"path": "a.py"}}
```
````

## MT5 ツール

- `mt5_compile`: `.mq5` を `MetaEditor64.exe /compile` でコンパイル。MQL5 フォルダ外のファイルは `MQL5/Experts/AperCode/` にコピーしてからコンパイルし、`.ex5` を元の場所にも戻す
- `mt5_backtest`: `terminal64.exe /config:tester.ini` でストラテジーテスターを実行し、HTML レポートから純利益・PF・DD などを抽出

いずれも `WINEPREFIX=~/.wine_mt5` を使う（config.yaml で変更可）。

## Claude API を使う場合

```yaml
llm:
  provider: anthropic
anthropic:
  model: claude-sonnet-4-5
```

環境変数 `ANTHROPIC_API_KEY` を設定して起動。

## ツールの追加

`codeagent/tools/` に新しいモジュールを作り、`register(Tool(...))` で登録して `tools/__init__.py` の `load_builtin()` に import を足す。
