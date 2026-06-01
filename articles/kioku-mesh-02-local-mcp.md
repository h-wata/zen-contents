---
title: "kioku-mesh を1台で動かして Claude Code / Codex CLI から使う"
emoji: "🪺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mcp", "claudecode", "codex", "python", "ai"]
published: false
---

:::message
本記事は **kioku-mesh 連載 第2回** です。前回はコンセプトとアーキテクチャを整理しました。今回は手元1台で `local` モードを動かし、Claude Code と Codex CLI に MCP として組み込んで「片方で保存 → もう片方で検索」までやります。前回はこちら → [kioku-mesh とは — AI コーディングエージェントの記憶を分散させない](#)
:::

## ゴール

- `pip install kioku-mesh` から `local` モードで初期化
- CLI で `save` / `search` / `get-memory` / `delete` / `gc` / `doctor` を一巡
- `kioku-mesh mcp install` で Claude Code と Codex CLI に登録
- 実際に Claude Code でメモを保存し、Codex CLI から検索できることを確認

外部ネットワークや別マシンは不要、1台で完結します。

## 前提

- Python 3.10+
- Linux（macOS は未検証、Windows は WSL2 推奨）
- Claude Code か Codex CLI のどちらか（できれば両方）がインストール済み
- [uv](https://docs.astral.sh/uv/) がインストール済み（未導入なら `curl -LsSf https://astral.sh/uv/install.sh | sh`）

## インストールと初期化

kioku-mesh は `kioku-mesh` と `kioku-mesh-mcp` の2コマンドを提供する CLI ツールなので、`uv tool install` で隔離環境にまとめて入れるのが扱いやすいです。

```bash
uv tool install kioku-mesh
kioku-mesh init --mode local
```

これで `~/.local/bin` 配下に2つのコマンドが配置され、後段の `mcp install` がそこを参照します。バージョンアップは `uv tool upgrade kioku-mesh`、入れ直しは `uv tool install --reinstall kioku-mesh` でできます。

:::message
`pip install kioku-mesh` でも動きますが、グローバル環境を汚さないため、また MCP クライアント側に渡す絶対パスがブレないために、本連載では `uv tool install` を推奨します。
:::

`init --mode local` は `~/.config/kioku-mesh/config.yaml` に `backend: local` を書き、SQLite だけで動く状態にします。`zenohd` も RocksDB も必要ありません。

セットアップ後に `doctor` でヘルスチェックしておくと安心です。

```bash
kioku-mesh doctor
```

パッケージは2つのコマンドを置きます。

- `kioku-mesh`: CLI（人間用）
- `kioku-mesh-mcp`: MCP クライアントから起動される stdio MCP server。直接叩く想定ではない

## CLI を一巡する

### save — 観測を保存する

```bash
kioku-mesh save "Chose Postgres over SQLite for analytics" \
  --memory-type decision \
  --importance 4 \
  --subject analytics-db
```

`--memory-type` は意味のあるカテゴリで分類するための値で、以下から選びます（`models.py:VALID_MEMORY_TYPES`）。

| memory_type | 使い所 |
| --- | --- |
| `note` | デフォルト。雑多なメモ |
| `decision` | 設計・方針の決定 |
| `bug` | 直したバグと原因 |
| `pattern` | コーディング規約・命名・構造のパターン |
| `config` | 設定変更とその理由 |
| `summary` | セッション終了時のまとめ |

`--importance` は 1〜5（デフォルト 2）。`--subject` は短いトピック名で、検索結果の見出しになります。
このほか `--tags`、`--source-files`、`--references`（PR/Issue 番号）、`--supersedes`（置き換える観測 ID）も指定できます。

### search — 検索する

```bash
kioku-mesh search "Postgres"
kioku-mesh search "" --memory-type decision --limit 20  # 空クエリ + 条件絞り込みも可
```

結果は `[memory_type][importance] timestamp content` の形で出ます。`--format json` で機械可読にもできます。

### get-memory / delete / gc

```bash
# 1件だけ完全な内容を取り出す（観測 ID は 32 桁 hex）
kioku-mesh get-memory <observation_id>

# 論理削除（tombstone を書く）
kioku-mesh delete <observation_id>

# 物理削除（30日より古い tombstone を回収）
kioku-mesh gc --retention-days 30
```

`delete` は tombstone を書くだけで、データはまだ消えていません。物理的に消したいときは `gc` を回します。これはメッシュ運用に切り替えた後にレプリケーション越しの削除整合性を取るための設計で、`local` モードでも挙動は同じです。

## エージェントごとの ID（環境変数）

kioku-mesh は誰が何を保存したかを区別するため、いくつかの ID を持ちます。MCP クライアント経由なら次節の `mcp install` がデフォルトを埋めてくれるので、CLI を素で使うときだけ意識すれば十分です。

| 環境変数 | 役割 |
| --- | --- |
| `MESH_MEM_AGENT_FAMILY` | エージェント系統（`claude` / `codex` など） |
| `MESH_MEM_CLIENT_ID` | クライアント名（`claude-code` / `codex-cli` など） |
| `MESH_MEM_SESSION_ID` | 任意。同一セッションをまとめたいときに |
| `MESH_MEM_STATE_DIR` | state ディレクトリ。デフォルトはユーザーデータディレクトリ配下 |
| `MESH_MEM_BACKEND` | `local` / `zenoh` を強制上書き |

## Claude Code に MCP として登録する

`kioku-mesh mcp install` がクライアントごとに必要な設定を書き込んでくれます。

```bash
kioku-mesh mcp install --client claude-code
```

このコマンドは内部的に次のことをやります（`mcp_install.py`）。

- `kioku-mesh-mcp` の絶対パスを解決
- `MESH_MEM_AGENT_FAMILY=claude`、`MESH_MEM_CLIENT_ID=claude-code` をデフォルトでセット
- Claude Code の MCP 設定にエントリを追加（登録名は既定で `kioku_mesh`）

中身を確認だけしたい場合は `--dry-run` を付けると、実行せず設定ブロックを表示します。既存登録を上書きしたいときは `--force`、追加の環境変数を入れたいときは `-e KEY=VALUE` を繰り返し指定できます。

```bash
# 実行せず内容だけ確認
kioku-mesh mcp install --client claude-code --dry-run

# 既存を上書き
kioku-mesh mcp install --client claude-code --force

# 追加 env を渡す
kioku-mesh mcp install --client claude-code -e MESH_MEM_SESSION_ID=morning-pair
```

## Codex CLI にも入れる

同じ要領で Codex CLI にも入れます。

```bash
kioku-mesh mcp install --client codex-cli
```

こちらはデフォルトで `MESH_MEM_AGENT_FAMILY=codex`、`MESH_MEM_CLIENT_ID=codex-cli` がセットされます。`local` モードの SQLite は両者で同じファイルを指すので、Claude Code が保存した観測を Codex CLI から読めますし、逆もできます。これが「ツール横断の共有メモリ」の最小形です。

Claude Desktop / Gemini CLI / ChatGPT Desktop など、その他のクライアントは `docs/mcp-clients.md` と `docs/multi-agent.md` を参照してください（v0.3 時点で `mcp install` のターゲットは Claude Code と Codex CLI の2つです）。

## 動作確認：エージェント横断で保存 → 検索

ここまでで、`local` モードの SQLite を Claude Code と Codex CLI が共有している状態になっています。簡単に動作確認しましょう。

1. Claude Code 側で、たとえばこう頼みます。
   > 「kioku-mesh に『今日は kioku-mesh を試している』を `note` として保存して」
2. Claude Code が `save_observation` 相当の MCP ツールを叩いて保存します。
3. Codex CLI 側で別セッションを開き、こう頼みます。
   > 「kioku-mesh で『kioku-mesh』を検索して」
4. Codex CLI が `search_memory` 相当のツールを叩き、1. で保存したエントリが返ってきます。

CLI からの確認なら、同じことを次の1コマンドで見られます。

```bash
kioku-mesh search "kioku-mesh"
```

エージェントが別物でも、`local` モードの SQLite を共有しているので結果が揃って出てきます。

## ここまでで「ツール横断」は達成

`local` モードはあくまで1台に閉じた SQLite です。同じマシン上にいる Claude Code / Codex CLI の間では memory が共有されますが、別マシンには伝播しません。

> In mesh mode the Zenoh/RocksDB store is the source of truth, and each host's SQLite is a fast local read cache rebuilt from it — not a separate copy you have to reconcile. `local` mode is a standalone, SQLite-only setup for a single machine, so its saves live only in that local store and are not replicated to a mesh.（README より）

マシン横断にしたいときは `hub` / `spoke` モードに切り替えます。`init --mode hub --force` でいつでも切り替えられるので、`local` で動作感覚をつかんでから次に進むのが安全です。

## 次回予告

第3回では、メッシュモードの裏側で動いている Zenoh + RocksDB + SQLite index の関係を整理します。「source of truth はどこで、なぜ SQLite が読みキャッシュなのか」「`zenohd` は何をしているのか」「`local` から `hub/spoke` に切り替えたとき、既存の保存はどうなるのか」あたりを扱います。

第4回で実際にメッシュを組むときの理解が一段深くなる回です。

## 参考リンク

- [kioku-mesh (GitHub)](https://github.com/h-wata/kioku-mesh)
- `docs/mcp-clients.md` / `docs/multi-agent.md`（リポジトリ内）
- 連載第1回: [kioku-mesh とは — AI コーディングエージェントの記憶を分散させない](#)
