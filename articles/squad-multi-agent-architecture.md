---

title: "tmux + Claude Code + Codex CLIでマルチエージェント開発チームを組んでみた"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["claudecode", "AIエージェント", "tmux", "マルチエージェント"]
published: true

---

:::message
本記事は Claude（AI）の支援を受けて執筆しています。内容は著者がレビュー・編集したうえで公開しています。
:::

Claude Code を使っていると、独立した複数の Issue を同時に進めたい、レビューは実装したのと別の AI モデルにやらせたい、という場面が出てきます。サブエージェントでは足りません。実装 → レビュー対応の再実装、という流れの途中でコンテキストが切れてしまうからです。コンテキストを抱えたまま独立して動くエージェントが複数欲しい。

そこで、tmux の pane に Claude Code と Codex CLI を並べ、YAML ファイルでタスクと報告をやり取りする仕組み **squad** を作りました。API で繋ぐ構成も考えましたが、SDK 差を吸収するレイヤーを自作・保守する手間に見合いません。手元で動いている CLI をそのまま並べれば、壊れても pane を1個 restart すれば戻ります。

この手の仕組みは前例が多いので、この記事では**人間が張り付かなくても止まらずに回すためのノウハウ**に絞って書きます。

@[card](https://github.com/h-wata/squad)

## 構成: Dispatcher 1体 + Worker 4体

```
Pane 0: Dispatcher (Claude)   タスク分配・進捗管理のみ担当
Pane 1-3: Worker 1-3 (Claude) 実装・調査
Pane 4: Terminal              汎用シェル
Pane 5: Aux-Shell              SSH等の汎用利用
Pane 6: Worker 4 (Codex)       設計相談・cross-review
```

```mermaid
flowchart LR
    User([ユーザー]) --> Dispatcher["Dispatcher (Pane 0)"]
    subgraph Workers["Worker (Pane 1-3 Claude / Pane 6 Codex)"]
        direction TB
        W1["Worker1"]
        W2["Worker2"]
        W3["Worker3"]
        W4["Worker4 (Codex)"]
    end
    Dispatcher -->|task YAML| Workers
    Workers -->|report YAML| Dispatcher
    Watch["watch.sh"] -.->|検知・通知| Dispatcher
    Dispatcher --> Dashboard["dashboard.md"]
```

タスク1本の流れはこうです。

```mermaid
sequenceDiagram
    participant D as Dispatcher
    participant Q as queue/projects/<project>
    participant W as Worker
    participant Dash as dashboard.md

    D->>Q: task YAML 書き込み
    D->>W: tmux通知
    W->>Q: report YAML 書き込み
    Q-->>D: report検知 (watch.sh)
    D->>Dash: 更新委譲
```

エージェント同士は直接会話せず、やり取りはすべて YAML ファイル越しです。

## Dispatcherはコードを書かない

Dispatcher の仕事は、来た指示をどのプロジェクトの・どの Worker のタスクに落とすか決めることだけです。task YAML を書いて tmux で通知し、report YAML を待つ。この3手を回すだけ。

実装まで Dispatcher に持たせた時期もありましたが、コードの中身がコンテキストを食って「誰にどこまで振ったか」を見失いました。Dispatcher の頭の中は割り当て表だけにしておきます。

タスクと報告はプロジェクトごとにディレクトリを分けます。混ぜると、別プロジェクトの report を読み違える事故が起きるからです。

```
queue/projects/<project>/
  tasks/worker1.yaml
  tasks/worker4.yaml     # Codex 向け
  reports/worker1_report.yaml
  reports/worker4_report.yaml
```

## 実装はClaudeに寄せ、Codexは設計とレビューに温存する

Codex の利用枠は Claude より先に尽きます。手数のかかる実装を Codex に流すと、肝心なときに枠がありません。実装は Claude（Worker 1-3）、Codex（Worker 4）は設計検討とレビュー。迷ったら Claude が既定値です。

PR は書いた本人と**逆の agent** にレビューさせます。Claude の PR は Codex に、Codex の PR は Claude に。同じモデルに書かせてレビューまでさせると、同じ癖で同じ穴を見逃します。この cross-review が通るまで、CI が緑でも merge しません。

## Workerの自己採点を禁止する（verify gate）

Worker の「実装できました」は当てになりません。放っておくと平気で「全部通りました」が返ってきます。

そこで `verify:` ブロックを持つタスクは、実装した Worker とは別の verifier サブエージェントが検証します。verifier が verify コマンドを worktree で実際に走らせ、受け入れ基準と突き合わせて pass / fail を返す。fail なら Worker が直して再検証、上限を超えたら人間行きです。completed を名乗れるのは別プロセスの検証を抜けた後だけ。これがあるとないとでは、報告の当てになり方がまるで違います。

## 止まったWorkerは常駐デーモンに拾わせる（watch.sh）

Claude Code の Worker には自動継続モードがなく、確認プロンプトが1つ出るとそこで黙って止まります。人間が張り付かない限り、止まったことに誰も気づきません。

監視を Claude Code の `/loop` にやらせてみましたが、トークンを食う上に抜け漏れが多く、実用に耐えませんでした。代わりに書いたのが常駐デーモン `watch.sh` です。report の出現を検知して Dispatcher に通知し、止まった Worker を見つけて知らせ、承認プロンプトに自動応答します。merge 済み worktree の掃除や Issue / PR / CI の低頻度巡回も兼ねます。Dispatcher がルーティングに集中できるのは、この常駐が裏で全部拾っているからです。

## Dispatcherのトークン節約は別記事で

Dispatcher には一番賢いモデルを割り当てているので、定型作業でトークンを浪費させないことが全体で一番効きます。task YAML の執筆と dashboard の更新をサブエージェントに切り出した話は、こちらにまとめました。

https://zenn.dev/hwata/articles/squad-dispatcher-token-diet

## リンク

- GitHub: https://github.com/h-wata/squad
