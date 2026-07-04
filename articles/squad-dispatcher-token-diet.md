---
title: "Claude CodeのサブエージェントでDispatcherのトークン消費を減らした"
emoji: "🎛️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claudecode", "claude", "ai", "llm", "マルチエージェント"]
published: true
---

Claude Code で複数のエージェントを並列に動かすと、司令塔（Dispatcher）のトークン消費がボトルネックになります。
自作のマルチエージェント環境 [squad](https://github.com/h-wata/squad) で、タスク指示書の執筆とダッシュボード更新をサブエージェントに任せて、Dispatcher のトークン消費を減らした話です。

## Fable の5時間リミットに毎回全滅していた

squad は tmux の1セッションに Dispatcher（司令塔）と Worker 1〜3（Claude）、Worker 4（Codex）が同居する構成です。
人間は Dispatcher にだけ曖昧な指示を投げ、Dispatcher がタスクに分解して各 Worker に振り分けます。

Dispatcher に Fable のような上位モデルを使うと、すぐに5時間のレート制限に当たります。
リミットは同一アカウントで共有なので、**Dispatcher が使い切ると Worker も含めて全部止まります**。
これが何度も起きました。

Dispatcher を Haiku などの安いモデルに落とす手も考えましたが、やめました。
曖昧な指示を解釈してタスクに変換するのが Dispatcher の役割なので、ここの賢さは削れません。
削るべきは別の場所です。

## トークンを食っていたのは事務作業だった

Dispatcher の消費を観察すると、大半は賢さの要らない作業でした。

1つ目はタスク指示書（task YAML）の執筆です。
worktree のセットアップ手順や検証コマンドまで書き込むと、実物で1件 100〜380 行。
発注のたびに Dispatcher がこれを書き下ろしていました。

2つ目はダッシュボード更新です。
Worker の報告受領や PR マージのたびに、Dispatcher が Markdown の表を Read して Edit する。
表の1行を書き換えるためだけに、毎回ファイル全体を読んでいました。

この2つをサブエージェントに任せることにしました。

## task-yaml-author: タスク指示書を書かせる

Claude Code のサブエージェントは `.claude/agents/` に Markdown を置くだけで定義できます。
1つ目が [task-yaml-author](https://github.com/h-wata/squad/blob/main/.claude/agents/task-yaml-author.md) です。

```md
---
name: task-yaml-author
description: Use this agent when the Dispatcher needs to author a detailed
  `queue/projects/<project>/tasks/worker{N}.yaml` ...
  Offloads heavy context (100-300 lines per YAML) from the Dispatcher.
tools: Bash, Read, Write, Grep, Glob
model: sonnet
---
```

どの Issue をどの Worker に振るかは Dispatcher が判断し、task-yaml-author はその結果を受け取って YAML 本文を書きます。
Dispatcher が渡すのは Issue 番号・担当 Worker・worktree キーなど数行だけ。
返ってくるのも YAML のパスとサマリ 3〜5 行だけです。

worktree 手順のテンプレートやブランチ運用ルールといったノウハウは、289 行あるエージェント定義ファイルに全部書いてあります。
この定義はサブエージェント側のコンテキストにだけロードされるので、Dispatcher は一度も読みません。

## dashboard-updater: 表の更新は Haiku で十分

2つ目が [dashboard-updater](https://github.com/h-wata/squad/blob/main/.claude/agents/dashboard-updater.md) です。
Markdown の表を書き換えるだけなので、モデルは Haiku を指定しています。
サブエージェントは作業ごとにモデルを選べるのが利点です。

```md
---
name: dashboard-updater
tools: Read, Edit, Write, Grep, Glob
model: haiku
---
```

Dispatcher はタスク ID・状態変化・PR URL など6項目を渡すだけ。
dashboard-updater が表を更新し、結果を 2〜4 行で返します。

```
✓ dashboards/squad.md: TASK-007 を Active → 完了タスク表に移動、担当 worker1、PR #9
✓ dashboard.md: Worker1 のステータスを 稼働中 → 待機中 に更新
```

定義ファイルには「dashboard 以外は編集しない」「GitHub へは投稿しない」という制約も書いてあります。
安いモデルに書き込み権限を渡すので、触れる範囲は定義側で狭めておきます。

## 結果: Dispatcher のトークン削減ができた

導入前後の transcript を比べると、Dispatcher 自身による dashboard の直接 Edit は 44 回 → 0 回になりました。
task YAML 全文（100〜380 行）と dashboard 全文も、Dispatcher のコンテキストに乗らなくなりました。
一度コンテキストに乗った内容は以降の全ターンで課金され続けるので、乗せないことの効果は累積します。

体感で一番大きいのはコンテキストの寿命です。
以前はタスク5本ごとに /compact（コンテキスト圧縮）が必要でしたが、いまは朝からタスク15本捌いて /compact なしで走り切れています。

委譲した分の執筆トークンはサブエージェント側の Sonnet / Haiku が使いますが、単価が安く、Dispatcher の消費とは切り離されています。

## まとめ

Dispatcher は賢いモデルのまま残し、タスク YAML の執筆と dashboard の更新をサブエージェントに任せました。
やったことは `.claude/agents/` に Markdown を2枚置いただけですが、Dispatcher のトークン削減ができ、/compact の頻度はタスク5本ごとから15本ノー compact まで減りました。
構成ファイルは [h-wata/squad](https://github.com/h-wata/squad) で公開しています。
