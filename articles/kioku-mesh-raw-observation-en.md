---
title: "Why I Didn't Make Graph DB the Source of Truth for an AI Agent's Long-Term Memory"
published: false
description: "How kioku-mesh keeps raw observations as the source of truth and treats Graph/Vector/Summary as derived views you can rebuild later."
tags: ai, memory, database, rust
canonical_url:
---

When you give an AI agent long-term memory, the first thing you get stuck on is where to store it. A Graph DB? A Vector DB? Summaries? Or is plain SQLite enough?

I ran into exactly this while building [kioku-mesh](https://github.com/h-wata/kioku-mesh), a memory backbone that lets coding agents like Claude Code and Codex share working context across sessions and machines. The goal is to make things like "the design decision we made last week," "the root cause of that bug," "why we changed this config value," or "an approach we tried once and dropped" recallable later, from a different agent or a different PC.

My conclusion: kioku-mesh does **not** make a Graph DB the source of truth for memory. Not "don't use a graph" — "don't make the graph canonical." That distinction is the whole point.

## The existing design: SQLite is a cache, not the truth

This decision didn't come out of nowhere. It lines up with how kioku-mesh was already built.

The source of truth in kioku-mesh is Zenoh's K/V. Observations flow through Zenoh as key-values and get persisted by a RocksDB backend. But scanning Zenoh/RocksDB on every query is expensive, so each host keeps a local SQLite index.

```
Zenoh K/V + RocksDB  ->  Source of Truth
SQLite local index   ->  Fast-search cache (derived)
```

SQLite is not the truth. If it gets corrupted or goes stale, it can be rebuilt from the Zenoh/RocksDB side. So kioku-mesh treats the SQLite index as a derived view and is designed to rebuild it on demand.

Everything below is just extending that same "separate the canonical store from its derived views" idea to graphs and embeddings.

## What a paper helped me sort out

The trigger was a paper, ["Are We Ready For An Agent-Native Memory System?"](https://arxiv.org/pdf/2606.24775). It frames agent memory not as plain RAG or vector search, but as a data management system.

There's no single representation for memory. Stream keeps the timeline as-is. Summary compresses. Vector is good at fuzzy semantic search. Graph is good at following relationships. Hybrid mixes them. None of these is universally best; each strength comes with a cost.

- Building a graph needs entity / relation extraction
- Vector depends on your embedding model
- Summary drops information at summarization time
- Hybrid gets complex in proportion

The thing that clicked for me: **which form you should store in depends on how you'll want to recall it later** — and at the moment you save something, you don't yet know what will matter.

## Don't make an interpretation the source of truth

Say you store an observation: "changed the Docker Compose network settings." At save time it looks like a plain config tweak. Later it might turn out to be the root cause of a bug that only happens on one machine, a workaround you tried before, or how you avoided a clash with another project.

If you normalize it into a graph at save time, you're locked into whatever entities and relations you could extract back then. Keep only a summary, and the information you dropped is gone for good. Make embeddings the canonical form, and you're stuck the day you want to switch embedding models.

So graphs, vectors, and summaries aren't the raw data itself — they're **an interpretation made at one point in time**. The scary part of long-term memory is letting that interpretation become the source of truth.

## Raw observations as the source of truth

So in kioku-mesh, the raw observation is kept as the source of truth. An observation is the working context an agent saves, e.g.:

```
For local dev, standardize ROS_DOMAIN_ID to 7.
Reason: 42 was clashing with another project.
```

You can turn this into a graph, into embeddings, into a summary, or index it for FTS. But the canonical copy stays the raw observation, untouched.

```
Raw Observation   =  Source of Truth
SQLite / FTS      =  Derived View
Embedding         =  Derived View
Graph             =  Derived View
Summary           =  Derived View
```

Build the graph or embeddings when you need them; rebuild if they don't work out; add another view when you need another way to recall. For long-term memory, that felt safer.

## This isn't a rejection of Graph DBs

To be clear: this isn't "don't use a Graph DB." It's "don't make a Graph DB the source of truth."

Questions like "which bug fix is this config change related to?", "what past decisions affect this file?", or "what constraint is behind this change of direction?" are exactly where a graph shines.

But that graph can be a derived view built from raw observations.

```
Raw Observation  ->  Entity / Relation extraction  ->  Graph View
```

This way you can change the extraction logic and the schema later, and switch to a different view if the graph turns out less useful than you hoped. It's the exact same position as the existing SQLite index: not the truth, just a view for recall. Graph, Embedding, and Summary all fit alongside SQLite as "derived views you can add later."

## Pick the view to match how you recall

What matters in long-term memory isn't choosing the perfect DB representation up front — it's **being able to change how you recall later**.

For searches over file names, function names, error strings, or config keys, FTS or a SQLite index is plenty. For fuzzy natural-language questions like "didn't we hit a bug like this before?", embeddings might win. For following relationships like "what fix had the same root cause as this bug?", a graph might win.

Which recall method works depends on the actual workload, and that changes. That's exactly why the source of truth shouldn't be pinned to one DB representation: keep it as the raw observation, and build derived views to match how you want to recall. With that setup, you can change your mind later.

## Wrapping up

After reading the paper, kioku-mesh's stance got a lot clearer:

> Don't avoid the Graph DB — just don't make it canonical. In long-term memory, keep the raw observation as the source of truth, and build graphs or embeddings later, to match how you recall.

Graph DBs, Vector DBs, and summaries are all useful, but they're not the memory itself — they're an interpretation at a point in time. The truth lives as a raw observation in Zenoh's K/V, persisted by RocksDB. SQLite is today's search cache, and if graphs or embeddings come in, they get the same derived-view treatment.

That, I think, is what lets an agent's long-term memory grow over time instead of calcifying.

---

*Originally written about [kioku-mesh](https://github.com/h-wata/kioku-mesh) — a local-first, peer-to-peer, MCP-native shared memory for AI coding agents.*
