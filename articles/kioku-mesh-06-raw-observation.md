---
title: "AI エージェントの長期記憶を Graph DB にしなかった理由 — kioku-mesh #6"
emoji: "🗂️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ai", "memory", "graphdb", "vectordb", "zenoh"]
published: true
---

:::message
本記事は Claude（AI）の支援を受けて執筆しています。内容は著者がレビュー・編集したうえで公開しています。
:::

:::message
本記事は kioku-mesh 連載の番外編です。前回までで local モードから mTLS まで一通り構築しました。今回はコードの手順ではなく、「長期記憶を何に保存するか」という設計判断について書きます。
:::

@[card](https://github.com/h-wata/kioku-mesh)

## どの DB に保存するか問題

AI エージェントに長期記憶を持たせようとすると、保存先の選択でまず手が止まります。Graph DB にするのか、Vector DB にするのか、要約を残すのか、それとも SQLite で十分なのか。

[kioku-mesh](https://github.com/h-wata/kioku-mesh) でもここは悩みどころでした。kioku-mesh は Claude Code や Codex のようなコーディングエージェントが、セッションや PC をまたいで作業文脈を共有するためのメモリ基盤です。「前に決めた設計方針」「あるバグの原因」「設定値を変えた理由」「一度試してやめた実装方針」——こうした後から効いてくる情報を、別のエージェントや別の PC からでも思い出せるようにすることを狙っています。

結論から言うと、kioku-mesh では Graph DB を長期記憶の Source of Truth にはしていません。**使わないのではなく、正本にはしない**、という判断です。

## もともとの構成: SQLite はキャッシュ

実はこの判断は、kioku-mesh の既存の構成と地続きです。

kioku-mesh の Source of Truth は Zenoh の K/V です。Observation を Zenoh の key-value として流し、RocksDB backend で永続化します。ただ検索のたびに Zenoh / RocksDB を広く scan するのは重いので、各ホストには SQLite の local index を持たせています。

```
Zenoh K/V + RocksDB  →  Source of Truth（正本）
SQLite local index   →  高速検索用キャッシュ（派生）
```

SQLite は正本ではありません。壊れても古くなっても、Zenoh / RocksDB 側から再構築できます。だから kioku-mesh では SQLite index を「派生ビュー」として扱い、必要なら rebuild する設計にしてあります（詳しくは [第3回](https://zenn.dev/h_wata/articles/kioku-mesh-03-architecture)）。

この「正本と派生を分ける」発想を、Graph や Embedding にもそのまま広げただけ、というのが今回の話です。

## 論文を読んで整理できたこと

きっかけは [“Are We Ready For An Agent-Native Memory System?”](https://arxiv.org/abs/2606.24775) という論文でした。エージェントのメモリを単なる RAG や Vector Search ではなく、データ管理システムとして捉える、という視点が示されています。

メモリの表現方法はひとつではありません。時系列をそのまま残す stream、圧縮する summary、曖昧な意味検索が得意な vector、関係をたどる graph、それらを混ぜた hybrid。どれかが常に最強ということはなく、得意分野と引き換えに弱点を抱えています。

- Graph にするには entity / relation extraction が要る
- Vector は embedding model に依存する
- Summary は要約の時点で情報を落とす
- Hybrid はそのぶん複雑になる

ここで腑に落ちたのは、**どの形で保存すべきかは「後からどう思い出したいか」に依存する**、という点でした。そして保存した瞬間には、後で何が重要になるかは分かりません。

## 解釈結果を正本にしない

たとえば「Docker Compose の network 設定を変更した」という Observation を保存したとします。保存時点ではただの設定変更に見えても、後から「特定の PC だけで起きる不具合の原因」「過去に試した workaround」「他プロジェクトとの衝突回避」として意味を持つかもしれません。

このとき保存時に Graph へ正規化してしまうと、そのとき抽出できた entity / relation に縛られます。Summary だけ残せば、落とした情報は二度と戻りません。Embedding だけを正本にすれば、後で embedding model を変えたくなったときに詰みます。

つまり Graph も Vector も Summary も、生データそのものではなく、**ある時点の解釈結果**です。長期記憶で怖いのは、この解釈結果を正本にしてしまうことだと思いました。

## Raw Observation を正本にする

そこで kioku-mesh では、Raw Observation をそのまま Source of Truth として残すことにしました。Observation はエージェントが保存する作業文脈そのもので、たとえばこういうものです。

```
ROS_DOMAIN_ID はローカル開発では 7 に統一する。
理由は、42 が別プロジェクトと衝突していたため。
```

これを Graph に変換することも、Embedding にすることも、Summary にすることも、FTS 用に index することもできます。でも正本は Raw Observation のまま動かしません。

```
Raw Observation   =  Source of Truth
SQLite / FTS      =  Derived View
Embedding         =  Derived View
Graph             =  Derived View
Summary           =  Derived View
```

Graph や Embedding は必要になったら作り、うまくいかなければ作り直す。別の想起方法が要るなら、別の view を足す。長期記憶としてはこのほうが安全だと考えました。

## Graph DB を否定しているわけではない

念のため。Graph DB を使わない、という話ではありません。Graph DB を**正本にしない**、という話です。

「この設定変更はどのバグ修正に関係するか」「このファイルに影響する過去の decision は何か」「ある方針変更の背景にある制約は何か」——こうした関係をたどる問いには、Graph がよく効きます。

ただ、その Graph は Raw Observation から作る派生ビューでいい。

```
Raw Observation  →  Entity / Relation extraction  →  Graph View
```

こうしておけば、抽出ロジックも schema も後から変えられますし、思ったほど効かなければ別の view に切り替えられます。これは既存の SQLite index と全く同じ立ち位置です。正本ではなく、想起のための view。Graph / Embedding / Summary は、SQLite と並ぶ「将来追加できる派生ビュー」の枠に収まります。

## 想起の仕方に合わせて view を選ぶ

長期記憶で大事なのは、保存時に完璧な DB 表現を選ぶことではなく、**後から想起の仕方を変えられること**だと思っています。

ファイル名・関数名・エラー文字列・設定キーのような検索なら FTS や SQLite index で十分です。「前に似たバグなかった？」のように自然言語で曖昧に探したいなら Embedding が効くかもしれない。「このバグと同じ原因の修正は？」のように関係をたどりたいなら Graph が効くかもしれない。

どの想起方法が効くかは実際の workload 次第で変わります。だからこそ Source of Truth を特定の DB 表現に固定せず、Raw Observation として残し、想起の仕方に合わせて派生ビューを作る。この構成なら、後から方針を変えられます。

## まとめ

論文を読んで、kioku-mesh の方針はかなりはっきりしました。

> Graph DB を使わないのではなく、正本にしない。長期記憶では Raw Observation を Source of Truth として残し、Graph や Embedding は想起の仕方に応じて後から作る。

Graph DB も Vector DB も Summary も便利ですが、それらは記憶そのものではなく、ある時点の解釈結果です。正本は Raw Observation として Zenoh K/V に置き、RocksDB で永続化する。SQLite は今の検索キャッシュで、Graph や Embedding を入れるとしても同じ派生ビューの扱いにする。

このほうが、エージェントの長期記憶を長く育てていけると考えています。

## 参考リンク

- 論文: [Are We Ready For An Agent-Native Memory System?](https://arxiv.org/abs/2606.24775)
- 連載第1回: [kioku-mesh とは](https://zenn.dev/h_wata/articles/kioku-mesh-01-intro)
- 連載第3回: [kioku-mesh の中身を理解する — Zenoh と RocksDB と SQLite index](https://zenn.dev/h_wata/articles/kioku-mesh-03-architecture)
- リポジトリ: [h-wata/kioku-mesh](https://github.com/h-wata/kioku-mesh)
