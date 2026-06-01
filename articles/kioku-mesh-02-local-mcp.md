---
title: "kioku-mesh を Claude Code と Codex CLI に MCP として繋ぐ"
emoji: "🪺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mcp", "claudecode", "codex", "python", "ai"]
published: false
---

:::message
本記事は Claude（AI）の支援を受けて執筆しています。内容は著者がレビュー・編集したうえで公開しています。
:::

:::message
本記事は **kioku-mesh 連載 第2回** です。前回はコンセプトとアーキテクチャを整理しました。今回は手元1台で `local` モードを動かし、Claude Code と Codex CLI に MCP として組み込んで「片方で保存 → もう片方で検索」までやります。前回はこちら → [kioku-mesh とは](https://zenn.dev/h_wata/articles/kioku-mesh-01-intro)
:::

@[card](https://github.com/h-wata/kioku-mesh)

## ゴール

- `uv tool install` から `local` モードで初期化
- `kioku-mesh mcp install` で Claude Code と Codex CLI に MCP server として登録
- Claude Code で保存した記憶を Codex CLI 側で引けることを確認

別マシンは不要、1台で完結します。

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
kioku-mesh doctor
```

`init --mode local` は `~/.config/kioku-mesh/config.yaml` に `backend: local` を書き、SQLite だけで動く状態にします。`zenohd` も RocksDB も要りません。`doctor` はヘルスチェック。

:::message
`pip install kioku-mesh` でも動きますが、グローバル環境を汚さないため、また MCP クライアント側に渡す絶対パスがブレないために、本連載では `uv tool install` を推奨します。
:::

入る2つのコマンドはこんな役割分担です。

- `kioku-mesh`: 人間用 CLI（メンテ・デバッグで時々叩く）
- `kioku-mesh-mcp`: MCP クライアントから起動される stdio MCP server。**実運用ではここしか使いません**

## Claude Code に MCP として登録する

`kioku-mesh mcp install` がクライアントごとの設定を書き込んでくれます。

```bash
kioku-mesh mcp install --client claude-code
```

このコマンドは内部で次のことをやります。

- `kioku-mesh-mcp` の絶対パスを解決
- `MESH_MEM_AGENT_FAMILY=claude` / `MESH_MEM_CLIENT_ID=claude-code` をデフォルトでセット（観測の作者を区別するため）
- Claude Code の MCP 設定にエントリを追加（登録名は既定で `kioku_mesh`）

中身だけ見たい場合は `--dry-run`、既存登録を上書きするときは `--force`、追加の環境変数を入れたいときは `-e KEY=VALUE` を繰り返し指定できます。

```bash
# 実行せず内容だけ確認
kioku-mesh mcp install --client claude-code --dry-run

# 既存を上書き
kioku-mesh mcp install --client claude-code --force

# 追加 env を渡す（例: セッション名を固定する）
kioku-mesh mcp install --client claude-code -e MESH_MEM_SESSION_ID=morning-pair
```

## Codex CLI にも入れる

同じ要領で Codex CLI にも入れます。

```bash
kioku-mesh mcp install --client codex-cli
```

こちらは `MESH_MEM_AGENT_FAMILY=codex` / `MESH_MEM_CLIENT_ID=codex-cli` がデフォルトでセットされます。`local` モードの SQLite は両者で同じファイルを指すので、Claude Code が保存した観測を Codex CLI から読めますし、逆もできます。

Claude Desktop / Gemini CLI / ChatGPT Desktop など、その他のクライアントは `docs/mcp-clients.md` と `docs/multi-agent.md` を参照してください（v0.3 時点で `mcp install` のターゲットは Claude Code と Codex CLI の2つ）。

## エージェントが呼ぶ MCP ツール

`mcp install` の後、エージェントからは kioku-mesh が提供する MCP tool が普通の tool として見えるようになります。主に使うのはこの2つです。

- `save_observation`: 観測を保存する
- `search_memory`: 観測を検索する

保存時のカテゴリ（`memory_type`）は次の6種類です。エージェントに「`decision` として保存して」と指示するときの語彙になるので、覚えておくと便利です。

| memory_type | 使い所 |
| --- | --- |
| `note` | デフォルト。雑多なメモ |
| `decision` | 設計・方針の決定 |
| `bug` | 直したバグと原因 |
| `pattern` | コーディング規約・命名・構造のパターン |
| `config` | 設定変更とその理由 |
| `summary` | セッション終了時のまとめ |

`importance`（1〜5）、`subject`（短い主題）、`tags`、`source_files`、`references`（PR/Issue 番号）なども MCP tool 引数として渡せます。エージェントが文脈を見て適切な値を埋めてくれるはずです。

## 動作確認：エージェント横断で保存 → 検索

`local` モードの SQLite を Claude Code と Codex CLI が共有している状態になっているので、片方で保存して、もう片方で検索すれば横断確認できます。

1. Claude Code 側で、こう頼みます。
   > 「kioku-mesh に『今日は kioku-mesh を試している』を `note` として保存して」
2. Codex CLI 側で別セッションを開き、こう頼みます。
   > 「kioku-mesh で『kioku-mesh』を検索して」
3. 1. で保存したエントリが返ってくれば OK

うまく見えないときの最後の手段として CLI から直接覗くこともできます。

```bash
kioku-mesh search "kioku-mesh"
```

## ここまでで「ツール横断」は達成

`local` モードはあくまで1台に閉じた SQLite です。同じマシン上の Claude Code / Codex CLI 間では memory が共有されますが、別マシンには伝播しません。

マシン横断にしたいときは `hub` / `spoke` モードに切り替えます。`init --mode hub --force` でいつでも切り替えられるので、`local` で動作感覚をつかんでから次に進むのが安全です。

## 次回予告

第3回では、メッシュモードの裏側で動いている Zenoh + RocksDB + SQLite index の関係を整理します。「source of truth はどこで、なぜ SQLite が読みキャッシュなのか」「`zenohd` は何をしているのか」「`local` から `hub/spoke` に切り替えたとき、既存の保存はどうなるのか」あたりを扱います。

第4回で実際にメッシュを組むときの理解が一段深くなる回です。

## 参考リンク

- [kioku-mesh (GitHub)](https://github.com/h-wata/kioku-mesh)
- `docs/mcp-clients.md` / `docs/multi-agent.md`（リポジトリ内）
- 連載第1回: [kioku-mesh とは](https://zenn.dev/h_wata/articles/kioku-mesh-01-intro)
