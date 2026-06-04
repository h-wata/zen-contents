---
title: "kioku-mesh #1 - 複数の PC・AI エージェント間で長期記憶を共有する OSS『kioku-mesh』を作った"
emoji: "🧠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["mcp", "claudecode", "codex", "zenoh", "ai"]
published: true
---

:::message
本記事は Claude（AI）の支援を受けて執筆しています。内容は著者がレビュー・編集したうえで公開しています。
:::

:::message
本記事は **kioku-mesh 連載 第1回** です。連載では、AI コーディングエージェントの記憶を複数のツール・複数のマシンで共有する OSS「kioku-mesh」を、手元1台で動かすところから、複数ホストでメッシュを組むところまで段階的に解説します。
:::

## kioku-mesh

[kioku-mesh](https://github.com/h-wata/kioku-mesh) は、複数台の PC 間・複数の AI エージェント間で、AI セッションの長期記憶を共有しておくためのツールです。`kioku`（記憶）の名のとおり、エージェントが決めたこと・直したバグ・気づいたこと・規約などを1つのプールに溜めておき、どの PC のどのエージェントからも引き出せるようにします。

@[card](https://github.com/h-wata/kioku-mesh)

## なぜ作ったか

自分は普段、自宅 PC とオフィス PC を 行き来しながら開発しています。そのとき、片方の PC で進めた作業を、もう一方のPCで続けようとすると、エージェントはそれまでのContextを覚えていません。同じ会話を最初からやり直すのが苦痛でした。

加えて、自分自身も履歴を忘れるというのが辛かったです。チャット履歴は、検索性は弱く、別マシンや別ツールに移ると拾えません。

もう一つ辛かったのが、複数エージェントで「コーディング」と「レビュー」を分けて動かすときです。サブのエージェントが毎回フォルダ全体を読みに行き、現状把握が終わるまで作業が進みません。

AI エージェント向け長期記憶ツールはいくつかあります。ただ、SaaS にデータを預けることが前提だったり、逆にLocalに閉じる前提だったりして、「自分の手元の複数台で、自分のデータのまま、LAN/VPN に閉じたまま共有する」というニーズには微妙に満たせませんでした。それなら自分で作ろう、というのが kioku-mesh のスタートです。

## kioku-mesh が解いていること

kioku-mesh の中身は、[Zenoh](https://zenoh.io/) ベースのメッシュです。Zenoh + RocksDB を source of truth に置き、その上に各ホストのローカルな SQLite 読みキャッシュを乗せています。

| 課題                            | kioku-mesh                                                                                                                                                |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 自宅 PC ↔ オフィス PC で文脈が引き継げない | Zenoh メッシュで save/search を全 PC に同期。新しい PC を1台 mesh に追加するだけで、過去に他の PC で save した memory も全部読める                                   |
| いつどんな決定をしたか思い出せない         | 観測に `memory_type=decision` / `subject` / `importance` を付けて save。後から `search` で時系列・主題で引ける                                                       |
| サブエージェントが毎回フォルダを読み直す   | MCP server (`kioku-mesh-mcp`) 経由で `search_memory` / `save_observation` を共有プールに直結。前のエージェントが残した文脈をそのまま引けるので、最初から本題に入れる |

ポイントは「自前 peer-to-peer メッシュ」「SaaS なし、中央アカウントなし」「MCP ネイティブ」の3つです。手元の信頼できるネットワーク（LAN / VPN / Tailscale）の中で完結します。

## アーキテクチャの全体像

メッシュモードで動かしたときの構造はこんな形です。

```mermaid
flowchart LR
  subgraph HostA["🖥️ Host A"]
    direction TB
    A1["Claude Code"]
    A2["Codex CLI"]
    AS[("SQLite index<br/>local read path")]
    AZ["zenohd<br/>Zenoh router + RocksDB<br/>(source of truth)"]
    A1 & A2 -->|"save / search"| AZ
    AZ -->|"subscriber · rebuild"| AS
    A1 & A2 -.->|"fast reads"| AS
  end
  subgraph HostB["🖥️ Host B"]
    direction TB
    B1["Codex CLI"]
    B2["Gemini CLI"]
    BS[("SQLite index<br/>local read path")]
    BZ["zenohd<br/>Zenoh router + RocksDB<br/>(source of truth)"]
    B1 & B2 -->|"save / search"| BZ
    BZ -->|"subscriber · rebuild"| BS
    B1 & B2 -.->|"fast reads"| BS
  end
  AZ <==>|"Zenoh mesh replication<br/>LAN / VPN / Tailscale · TCP 7447"| BZ
```

読み方は3つです。

1. source of truth は Zenoh + RocksDB。ホストごとに `zenohd` が動いて RocksDB に observation を保存し、ホスト間はこの層が Zenoh のレプリケーションで同期する。
2. 読みは SQLite index。各ホストの SQLite は RocksDB から構築されるローカルな読み専用インデックスで、`search` はここで完結する。同期で揃えるコピーではなく、いつでも捨てて作り直せるキャッシュ。
3. エージェントは MCP 経由。`kioku-mesh-mcp` 越しに save/search するだけで、エージェント側に Zenoh の知識は要らない。

なお1台だけで使う `local` モードでは Zenoh も RocksDB も使わず、SQLite 単体で完結します。この場合の保存はあくまでローカル止まりで、メッシュには載りません。

## モードの使い分け

kioku-mesh は実用上3つのモードを覚えれば十分です。

| モード  | 使い所                                   | 永続化  | 追加サービス |
| ------- | ---------------------------------------- | ------- | ------------ |
| `local` | 1台で完結させたい                        | SQLite  | なし         |
| `hub`   | このマシンを常時起動のメッシュハブにする | RocksDB | `zenohd`     |
| `spoke` | hub に接続する側                         | RocksDB | `zenohd`     |

おすすめの始め方は、まず `local` で動かして MCP 接続まで体験する、必要になったら `hub` + `spoke` に切り替える、という順番です。`init --mode <mode> --force` でいつでも切り替えられます。

:::message
このほかに Zenoh 自体の動作確認用として `localhost` モードもありますが、in-memory・1台限定で実運用フローには乗らないため、本連載では扱いません。
:::

## まずは触ってみる

`uv tool install` で2つのコマンド（`kioku-mesh` と `kioku-mesh-mcp`）が入ります。

```bash
uv tool install kioku-mesh
kioku-mesh init --mode local
kioku-mesh save "Today lunch is Onigiri"
kioku-mesh search "Onigiri"
kioku-mesh mcp install --client claude-code
```

`mcp install` の後は、エージェント側からは `save_observation` / `search_memory` が普通の MCP tool として呼べるようになります。Codex CLI など他クライアントも同じ形で繋げます。

詳しいコマンド・引数の使い方と、MCP からの動作確認は第2回でやります。

## 制約とセキュリティの前提

メッシュを公開する前に、現状の制約をはっきりさせておきます。

- バージョンはまだ `0.x` で実験的です。今後破壊的な変更があり得ます。
- 開発は Linux 中心です。macOS / Windows は十分な検証ができていません（Windows は WSL2 推奨）。
- **信頼できるネットワーク上で動かす前提**の設計です。インターネットに公開してしまうと `7447/tcp` 経由で Zenoh の key-value が外から見える状態になります。LAN / VPN / Tailscale など閉域網での利用を強く推奨します。

それでも足りないケース（Tailscale ACL のミス対策、同居人がいる LAN で動かしたい等）に向けて、ピア間 mTLS を有効化する手段も用意しています。詳細は連載第5回で扱います。

## 連載で扱うこと

1. **第1回（本記事）**: kioku-mesh を作った動機、全体像、最初の触り方
2. **第2回**: `local` モードで一通り動かし、Claude Code / Codex CLI に MCP として組み込む
3. **第3回**: メッシュの中身（Zenoh + RocksDB + SQLite index）の役割整理
4. **第4回**: hub と spoke を立てて複数ホストでメッシュを組む
5. **第5回（任意）**: mTLS でピア間通信を締める

複数 PC・複数エージェント間でエージェントの記憶を共有できる状態をゴールにします。

## リンク

- PyPI: <https://pypi.org/project/kioku-mesh/>
- GitHub: <https://github.com/h-wata/kioku-mesh>
- Demo: <https://github.com/h-wata/kioku-mesh/blob/main/docs/assets/demo.gif>
- 影響を受けたプロジェクト: [engram](https://github.com/Gentleman-Programming/engram) / [claude-mem](https://github.com/thedotmack/claude-mem)
