---
title: "司令塔は賢く、手足は安く。Claude CodeサブエージェントでDispatcherのトークンを節約する"
emoji: "🎛️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claudecode", "claude", "ai", "llm", "マルチエージェント"]
published: true
---

Claude Code で複数のエージェントを並列に動かす「マルチエージェント運用」を始めると、すぐにトークン消費の壁に当たります。
この記事では、tmux 上で司令塔（Dispatcher）と作業者（Worker）を走らせている自作のマルチエージェント環境 [squad](https://github.com/h-wata/squad) を題材に、司令塔のモデルを賢いまま維持しつつトークン消費を抑える方法を紹介します。
結論を先に言うと、**司令塔の頭脳をダウングレードするのではなく、タスク指示書の執筆やダッシュボード更新といった「事務作業」をサブエージェントに剥がす**のが答えでした。

## Fable の5時間リミットに毎回全滅していた

squad の構成はシンプルで、tmux の1つのセッションに Dispatcher（司令塔）と Worker 1〜3（Claude）、Worker 4（Codex）が同居しています。
人間は Dispatcher にだけ曖昧な指示を投げ、Dispatcher がそれをタスクに分解して各 Worker に振り分けます。

問題はトークンでした。
Dispatcher に Fable のような上位モデルを使っていると、すぐに5時間のレート制限に当たってしまう。
タスクを走らせ始めても、じきにリミットに達して止まる。
そして厄介なのは、止まるのが Dispatcher だけではないことです。
リミットは同一アカウントで共有されているので、**Dispatcher が使い切ると Claude Code のエージェントが全部止まります**。
Worker がせっかく並列で作業していても、司令塔の浪費が全員を道連れにする。
これが何度も起きました。

## Dispatcher を安いモデルにする案は試すまでもなく捨てた

最初に考えるのは「Dispatcher 自体を Haiku などの安いモデルにすればいいのでは」という案です。
Dispatcher のモデルは賢くあるべきか、安いモデルで十分か。これはずっと悩んでいたポイントでした。

結論としてこの案は、実験するまでもなく捨てました。
理由は要件との矛盾です。
私が Dispatcher に投げたいのは「現在のリポジトリで導入で躓きそうな点を探してフォローして」のような曖昧な指示です。
何を調べるか、どのファイルを見るか、どの Worker に振るかは全部 Dispatcher に考えてほしい。
曖昧な指示を受け取り、文脈を補い、適切な粒度のタスクに変換するのが Dispatcher の存在意義そのものなので、そこを担うモデルの知能は削れません。
安いモデルで試して失敗してから学ぶまでもなく、要件から即断できる話でした。

つまり削るべきは「頭脳」ではない。
では何を削るのか。

## 消費の正体は「頭脳」ではなく「事務作業」だった

Dispatcher のトークン消費を観察すると、大半は賢さの要らない作業に使われていました。

1つ目はタスク指示書（task YAML）の執筆です。
squad では Worker への指示を YAML ファイルで渡しますが、worktree のセットアップ手順、ブランチ運用、検証コマンド、報告様式まで書き込むと、実物で1件 36〜381 行になります。
重いプロジェクトだと 273〜381 行。
これを Dispatcher が発注のたびに自分で書き下ろしていたので、1回の発注で数百行分の出力トークンが飛んでいました。

2つ目はダッシュボード更新です。
squad は進捗をプロジェクト別の dashboard（Markdown の表）で管理していて、Worker からの報告受領、タスク発注、PR マージのたびに Dispatcher が該当ファイルを Read して Edit していました。
dashboard 群は index が 53 行、プロジェクト別が 88〜209 行。
表の1行を書き換えるためだけに、毎回ファイル全体を読む。
知能はまったく要らないのに、イベントのたびに確実にトークンを食う作業です。

曖昧な指示の解釈には賢さが要る。
しかし YAML の清書と表の更新には要らない。
この線引きが見えたので、後者をサブエージェントに剥がすことにしました。

## task-yaml-author: タスクの「執筆」を剥がす

1つ目のサブエージェントが [task-yaml-author](https://github.com/h-wata/squad/blob/main/.claude/agents/task-yaml-author.md) です。
Claude Code のサブエージェントは `.claude/agents/` に Markdown を置くだけで定義でき、Dispatcher からは Task tool 経由で呼び出せます。

分業のポイントは、**判断と執筆を分離した**ことです。
どの Issue をどの Worker に振るか、worktree をどう切るか、なぜその割り当てなのか。
ここまでは Dispatcher の判断です。
task-yaml-author はその判断結果だけを受け取って、YAML の本文を書きます。

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

Dispatcher が渡すのは project、worker、task type、対象の Issue/PR 番号、worktree キー、routing_reason（なぜこの割り当てか）だけ。
数行の入力から、273〜381 行の指示書が生成されます。

もう1つ効いているのが、**ノウハウの置き場所が Dispatcher のコンテキストから外に出た**ことです。
task-yaml-author の定義ファイルは 289 行あり、worktree セットアップ手順のテンプレート、ブランチ運用ルール、Claude タスクと Codex タスクの様式の違いといったノウハウが全部そこに書いてあります。
以前はこの種の知識を Dispatcher 自身の指示書に持たせる必要がありました。
今は Dispatcher はこの 289 行を一度も読みません。
サブエージェントの定義ファイルは、呼び出されたサブエージェント自身のコンテキストにだけロードされるからです。

返ってくるのは生成した YAML のパスと 3〜5 行のサマリだけ。
Dispatcher はそれを確認して、tmux 経由で Worker に「タスクがあるよ」と通知するだけで発注が完了します。

## dashboard-updater: 更新作業を Haiku に落とす

2つ目が [dashboard-updater](https://github.com/h-wata/squad/blob/main/.claude/agents/dashboard-updater.md) です。
こちらは task-yaml-author よりさらに割り切っていて、モデルに Haiku を指定しています。

```md
---
name: dashboard-updater
tools: Read, Edit, Write, Grep, Glob
model: haiku
---
```

やることは Markdown の表の行を追加・移動・書き換えするだけなので、賢さは要りません。
Dispatcher の頭脳は削れませんが、サブエージェントは作業ごとに適正なモデルを選べます。
これがサブエージェント分離のもう1つの利点で、**「Dispatcher は賢く、手足は安く」をモデル指定のレベルで実現できます**。

Dispatcher が渡す入力は6項目だけです。

- project: プロジェクト名
- task_id: 対象タスク ID
- worker: 担当 Worker
- new_status: 状態変化（発注 / 進行中 / 完了 / merge 済み）
- artifacts: PR URL や commit SHA
- timestamp: 日時

dashboard-updater は該当する dashboard を読んで表を更新し、「更新した行の要約」を 2〜4 行だけ返します。

```
✓ dashboards/squad.md: TASK-007 を Active → 完了タスク表に移動、担当 worker1、PR #9
✓ dashboard.md: Worker1 のステータスを 稼働中 → 待機中 に更新
```

Dispatcher 側から見ると、イベント要約1行を投げて確認2〜4行を受け取るだけ。
計 657 行ある dashboard 群を、Dispatcher はもう読みません。

定義ファイルには「dashboard 以外のファイルは一切編集しない」「GitHub への投稿はしない」という行動制約も書いてあります。
安いモデルに書き込み権限を渡すので、触ってよい範囲を定義側で狭めておくのは安全のための実務的なポイントです。

## サブエージェント以外の小技も効く

サブエージェント化は「書く量」の削減でした。
あわせて「読む量」も削っています。squad の [PR #8](https://github.com/h-wata/squad/pull/8) で入れた小技です。

まず Worker の報告様式です。
以前は Worker が自由に書いた長い報告を Dispatcher が全部読んでいました。
今は報告 YAML の summary を 10 行以内に制限し、詳細は別ファイルに書いて details_path でパスだけ示す方式にしました。
Dispatcher は summary だけ読んで、必要なときだけ詳細を開きます。

次にセッション復元です。
Dispatcher のセッションを立ち上げ直すとき、以前は全プロジェクトの dashboard と過去記憶を広めに読み込んでいました。
今はアクティブなプロジェクトの dashboard だけを読み、記憶検索の limit も 30 件から 10 件に絞っています。

どれも地味ですが、Dispatcher は長時間動き続けるプロセスなので、イベントごと・再起動ごとの固定費の削減がそのまま稼働時間に効いてきます。

## 実測: 総トークンは減らない。減るのは司令塔の「寿命の消費」だ

効果を実測してみると、予想と少し違う結果が出ました。

まず狙いどおりだった部分。
稼働中セッションの transcript を分析すると、dashboard-updater 導入前は Dispatcher 自身が dashboard を直接 Edit した回数が 44 回。
導入後は 0 回です。
節約の実体は Edit の回数そのものではなく、dashboard 2ファイル分（約 12.6KB）と task YAML（1本 100〜300 行）が Dispatcher のコンテキストに一度も乗らなくなったことにあります。
一度コンテキストに乗った内容は、以降の全ターンでキャッシュ込みで課金され続けるので、乗せないことの累積効果が大きい。

一方で、委譲はタダではありません。
ある日の実測では、task-yaml-author が1回あたり 28k〜33k トークンで約8回、dashboard-updater が1回あたり 23k〜27k トークンで約9回。
サブエージェント側で計 40〜50 万トークンを使っていました。
つまり**システム全体の総トークン量は、むしろ増えている可能性すらあります**。

それでもこの分業は成立しています。
増えた 40〜50 万トークンは Sonnet と Haiku の安いトークンで、代わりに上位モデルの Dispatcher は YAML 全文も dashboard 全文も一切抱えません。
効果が一番はっきり出たのはコンテキストの寿命です。
以前はタスク5本ごとに /compact（コンテキスト圧縮）が必要だったのが、この日は朝からタスク15本近く捌いて /compact ゼロでした。

まとめると、効くのは「総トークン量」ではなく、**$コストと Dispatcher の寿命（compact 頻度）**です。
書き取り仕事のトークンを、安い別の担当に払わせる。
それがこの構成の実態です。

## まとめ: 司令塔は賢く、手足は安く

マルチエージェント運用の司令塔は、曖昧な指示を解釈するために賢いモデルであるべきです。
その代わり、賢さの要らない定型作業 — タスク指示書の執筆とダッシュボード更新 — をサブエージェントに剥がす。
総トークン量は相殺気味でも、賢いモデルのコンテキスト消費と compact 頻度、そして $コストには明確に効きます。

構成ファイルはすべて [h-wata/squad](https://github.com/h-wata/squad) で公開しています。
司令塔は賢く、手足は安く。
`.claude/agents/` に Markdown を2枚置くだけで、この分業は今日から始められます。
