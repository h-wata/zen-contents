---
title: "kioku-mesh で AI との対話を振り返る — 日報・週報を自動生成する"
emoji: "📋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mcp", "claudecode", "kiokumesh", "ai", "生産性"]
published: true
---

:::message
本記事は Claude（AI）の支援を受けて執筆しています。内容は著者がレビュー・編集したうえで公開しています。
:::

みなさん、日々の振り返りはできていますか？
AI に相談することが増えている中で、どんなことを相談して決めていったか、今週どんなことが進んでいったか追いづらくなっているとおもいます。

複数のターミナルで並列に作業を行っていると作業密度が上がり、自分の中でもどのようなコンテキストで作業を行っていたかわからなくなると思います。

チャット履歴を遡れば書いてはあるものの、複数の PC や複数のツールをまたぐと途端に追えなくなります。kioku-mesh を使うと、AI エージェントが残した観測を横断検索できるので、「先週の decision を全部出して」「今日バグを直した記録を拾って」といった振り返りが1コマンドでできるようになります。

## AI との作業記録が追いづらい理由

プルリクやイシューは GitHub に残ります。しかし、その過程で AI に何を相談したか・どんな議論をしたか・なぜその実装方針にしたかは、チャット履歴にしか残りません。

- 別の PC に移ると履歴が引けない
- 複数エージェント（Claude Code + Codex CLI など）を使うと記録が分散する
- 「あのとき何でこう決めたんだっけ」が後から追えない

kioku-mesh は、エージェントが `save_observation` で決定・バグ修正・気づきを保存しておくことで、この「AI との対話の記録」を横断検索できる共有プールを作ります。

## search_memory で振り返る

kioku-mesh の `search_memory` には `since_iso` パラメータがあり、期間を指定して観測を取得できます。

```bash
# 今日の観測を確認
kioku-mesh search --since today

# 先週の decision だけ絞り込む
kioku-mesh search --since "2026-05-26" --memory-type decision
```

MCP 経由で Claude Code から呼ぶと、こんな感じで記録が出てきます。

```
[decision][3] 2026-06-03T05:16:00Z (kioku-mesh) Dockerfile を Glama 登録用に追加する方針
[bug][4]      2026-06-02T03:02:00Z (kioku-mesh) zenohd 起動前に port 7447 の競合チェックを追加
[pattern][2]  2026-06-01T14:35:00Z (kioku-mesh) PR / commit に絵文字を使わない
```

これを project 別・memory_type 別に整理すれば、そのまま日報・週報になります。

## mesh-mem-worklog スキルで自動化する

自分は、毎回手で整形するのは面倒なので、Claude Code のスキルとして自動化しています。

「kioku-mesh から今日の作業ログ作って」と話しかけると：

1. 対象期間を確認（または自動判定）
2. `search_memory` で観測を取得
3. project 別・memory_type 別（decision → bug → pattern → note の順）にグループ化
4. Obsidian の Daily ノートに追記

※Obsidianの部分は個人のDocumentsのPathでも良いです。

実際の出力はこんな形になります。

```markdown
## 10:30 kioku-mesh work log

### kioku-mesh

**decision**
- 09:14 Glama バッジ対応は awesome-mcp-servers マージのための通行手形と割り切る

**bug**
- 08:47 Reddit スパムフィルター対策: 新規アカウントでのリンク付き投稿は弾かれやすい

### _global

**pattern**
- 11:02 OSS 告知は Gist + 記事のセットで検索流入を作る

#worklog #kioku-mesh
```

GitHub の PR/Issue には「何をしたか」が記録され、kioku-mesh には「なぜそうしたか・AI とどう議論したか」が記録される。この2つが揃うと、後から開発の経緯をかなり正確に追えるようになります。

スキルの SKILL.md はこちら：

@[card](https://gist.github.com/h-wata/943f5e9c9e0e7309b4ade63f9f3979bc)

## 使い方

### 1. kioku-mesh のセットアップ

kioku-mesh が未導入の場合は第1回・第2回を参照してください。

@[card](https://zenn.dev/h_wata/articles/kioku-mesh-01-intro)

### 2. スキルを配置する

Gist から SKILL.md を取得して、Claude Code のスキルディレクトリに置きます。

```bash
mkdir -p ~/.claude/skills/mesh-mem-worklog
curl -sL https://gist.github.com/h-wata/943f5e9c9e0e7309b4ade63f9f3979bc/raw/SKILL.md \
  -o ~/.claude/skills/mesh-mem-worklog/SKILL.md
```

あとは Claude Code に「kioku-mesh から今日の作業ログ作って」と話しかけるだけです。

:::message
Obsidian のパス（`/home/User/Documents/my_obsidian/Daily/`）は SKILL.md 内に直書きされています。自分の環境に合わせて書き換えてください。
:::

### 3. エージェントに保存させる

振り返りの質は、エージェントがどれだけ観測を保存しているかに依存します。Claude Code なら `kioku-mesh` MCP が有効になっていれば、決定・バグ修正・発見を都度 `save_observation` で記録してくれます。

## 将来的にやりたいこと

現状は個人の複数 PC 間での共有ですが、**小規模チームでメッシュを共有する**のが次のステップだと考えています。チームメンバーそれぞれが `spoke` としてメッシュに参加すれば、誰がいつどんな決定をしたかをチーム全体で引けるようになります。プルリクのコメントには書かれないような「なぜこの設計にしたか」の文脈が、チーム全体の共有知になるイメージです。

## リンク

- kioku-mesh GitHub: <https://github.com/h-wata/kioku-mesh>
- mesh-mem-worklog SKILL.md: <https://gist.github.com/h-wata/943f5e9c9e0e7309b4ade63f9f3979bc>
- 連載第1回: <https://zenn.dev/h_wata/articles/kioku-mesh-01-intro>
- 連載第2回: <https://zenn.dev/h_wata/articles/kioku-mesh-02-local-mcp>
