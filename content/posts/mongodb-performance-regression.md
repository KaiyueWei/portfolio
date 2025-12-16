---
title: "Debugging a MongoDB Performance Regression with perf, Flamegraphs, and gdb (4.2.1 vs 4.0.13)"
date: '2025-12-16T00:00:00-07:00'
draft: false
categories: ["Performance", "Database"]
tags: ["MongoDB", "Debugging", "perf", "gdb", "Flamegraphs"]
description: "A deep dive into debugging a write-heavy performance regression in MongoDB 4.2.1 using perf, flamegraphs, and gdb."
---

I recently chased down a write-heavy performance regression that showed up when running the same workload on MongoDB 4.2.1 versus MongoDB 4.0.13. The punchline: the “slow” version spent a surprising amount of CPU time rebuilding WiredTiger row-store keys during eviction-driven reconciliation. With a targeted fix to key handling, the regression disappeared.

This post walks through the debugging approach end-to-end: how I reproduced the issue, what the flamegraphs revealed, how I used gdb to confirm the execution path, and why the root cause points to a specific key-buffer reset.

## TL;DR

*   The workload (many secondary indexes + concurrent updates) creates heavy eviction/reconciliation pressure in WiredTiger.
*   In MongoDB 4.2.1, CPU time shifts dramatically into:
    *   `__wt_row_leaf_key_copy`
    *   `__wt_row_leaf_key_work`
*   In MongoDB 4.0.13, that cost is essentially absent; the equivalent “key work” shows up elsewhere (e.g. `__rec_cell_build_leaf_key`) and is much cheaper overall.
*   Evidence from stack traces and conditional breakpoints shows the key-copy path is hit repeatedly across threads during eviction/reconcile.
*   **The likely culprit:** an incorrect reset of `key->size` in `__wt_row_leaf_key_work`, invalidating cached decompressed keys and forcing repeated key reconstruction.

## The Setup

I reproduced this in a clean environment (AWS EC2, Ubuntu 20.04), with the key goal being: make both versions run the same workload shape, then profile only the “interesting” phase.

The workload has three phases:

1.  **Start a clean mongod** with a fixed WiredTiger cache size.
2.  **Insert phase**
    *   Create many compound secondary indexes across 5 collections.
    *   Bulk insert 1M documents per collection using 5 concurrent threads.
3.  **Update phase (the trigger)**
    *   Run 5 concurrent threads, each repeatedly calling `updateMany` with an `_id` modulo predicate:
        ```javascript
        query: { _id: { $mod: [100, i] } }
        update: { $inc: { a: 1 } }
        ```

This shape matters: lots of secondary indexes amplify per-update work, and concurrency increases dirty-page production—together pushing WiredTiger into frequent eviction and reconciliation.

## Reproducing “Fast” vs “Slow”

I ran the same script against two builds by switching `MONGO_HOME`.

**Fast run (MongoDB 4.0.13):**

```bash
export MONGO_HOME="$HOME/work/mongo-4.0.13"
bash ./repro/repro_perf_v3.sh
```

**Slow run (MongoDB 4.2.1):**

```bash
export MONGO_HOME="$HOME/work/mongo-4.2.1"
bash ./repro/repro_perf_v3.sh
```

The script profiles the server process during the update phase using `perf`, then generates a flamegraph. The output artifacts are:

*   `results/flamegraph_mongo-4.0.13.svg`
*   `results/flamegraph_mongo-4.2.1.svg`

## First Clue: Flamegraph Deltas

Flamegraphs are great for answering one question quickly: where did the CPU time go?

### What “slow” looks like (4.2.1)

![Flamegraph MongoDB 4.2.1](/images/flamegraph_mongo-4.2.1.svg)

The slow run is dominated by WiredTiger eviction and reconciliation, and—critically—by row-store key copying/materialization:

*   `__wt_evict_thread_run` dominates sampled execution (eviction thread work).
*   Within reconciliation:
    *   `__wt_reconcile` and `__wt_rec_row_leaf` are major contributors.
*   Row-key handling is extremely hot:
    *   `__wt_row_leaf_key_copy` is a top contributor (~23% in my capture).
    *   `__wt_row_leaf_key_work` is also a top contributor (~22%).

At this point, the story becomes: “eviction/reconciliation is expected under write pressure, but why are we spending so much time copying keys?”

### What “fast” looks like (4.0.13)

![Flamegraph MongoDB 4.0.13](/images/flamegraph_mongo-4.0.13.svg)

The fast run still spends time in eviction/reconcile (not surprising), but does not burn CPU in `__wt_row_leaf_key_copy`:

*   `__wt_row_leaf_key_copy` is negligible (~0.08%).
*   Key work shows up instead as `__rec_cell_build_leaf_key` (noticeable, but not pathological).

That difference is the smoking gun: the regression isn’t “MongoDB updates are slower in general”—it’s that 4.2.1 takes a substantially more expensive key path inside WiredTiger reconciliation.

## Turning Samples into an Execution Path

Flamegraphs tell you what is hot, but not always why it’s hot. To tie the hotspot to a real request path, I used `gdb` and captured stack traces during the update phase.

A representative call chain from the debug session shows the hotspot sits directly on the update commit path:

```text
MongoDB update execution:
  mongo::performUpdates
  mongo::UpdateStage::transformAndUpdate
  mongo::WiredTigerRecoveryUnit::_txnClose / commitUnitOfWork

WiredTiger commit + eviction/reconcile:
  __wt_txn_commit
  __wt_cache_eviction_worker
  __wt_evict / __evict_page
  __wt_reconcile / __wt_rec_row_leaf
  __wt_row_leaf_key_copy
```

This mattered because it ruled out a bunch of red herrings (client-side overhead, query parsing, planner differences, etc.). The slowdown is firmly inside “commit triggers eviction, eviction triggers reconcile, reconcile does expensive key work.”

## Confirming It’s Not a One-Off: Conditional Breakpoints

To test whether the key-copying path was rare or pervasive, I set a conditional breakpoint on `__wt_row_leaf_key_copy` that only triggers when the destination key buffer is empty:

```gdb
stop only if key->size==0 && key->data==0
```

This breakpoint hit repeatedly, across different threads (foreground mongod threads and background threads). That was important evidence that the system is frequently reaching a state where it has to instantiate/copy keys from scratch, rather than reusing an existing representation.

In other words: not a single pathological page, but a systemic behavior under this workload.

## What’s Actually Happening (Interpretation)

Here’s the high-level mechanism that fits all the evidence:

1.  The update workload creates a large amount of dirty data quickly (secondary indexes amplify it).
2.  WiredTiger starts evicting pages aggressively to keep cache under control.
3.  Eviction requires reconciliation: turning in-memory state into on-disk page images.
4.  During reconciliation of row-store leaf pages, WiredTiger needs access to keys.
5.  In 4.2.1, reconciliation frequently takes a path that materializes and copies row keys (`__wt_row_leaf_key_work` → `__wt_row_leaf_key_copy`) far more often than in 4.0.13.
6.  That additional key work becomes the dominant CPU cost, slowing eviction progress—and because eviction is on the critical path for commits under pressure, overall throughput collapses.

## Root Cause (Evidence-Based Hypothesis)

The regression in 4.2.1 appears to be caused by an incorrect reset of `key->size` in `__wt_row_leaf_key_work`, which invalidates cached decompressed keys. Once invalidated, WiredTiger has to rebuild full keys repeatedly during eviction/reconciliation, which matches:

*   the flamegraph shift (`__wt_row_leaf_key_copy`/`__wt_row_leaf_key_work` becoming top hotspots),
*   the stack traces showing the path is eviction→reconcile during update commits,
*   and the conditional breakpoint repeatedly catching `key->size==0 && key->data==0`.

### The fix

Remove the incorrect reset so cached decompressed keys remain reusable; this restores key reuse and eliminates the extra key reconstruction work during eviction.

## Lessons Learned / Debugging Checklist

A few takeaways I’d reuse for the next performance regression:

*   **Profile the smallest “interesting” phase.** I only profiled the update phase; the insert phase just sets up pressure.
*   **Compare flamegraphs across versions.** The delta often points straight at the subsystem that changed.
*   **Use gdb to connect hotspots to real request paths.** It’s the fastest way to avoid guessing.
*   **Conditional breakpoints are underrated.** They can tell you whether a suspicious state is rare or happening constantly under load.
*   **Eviction costs can become user-visible.** Under high write pressure, eviction/reconcile work is often on the critical path for commit latency and throughput.

## Artifacts / Repro Notes

The full reproduction scripts and logs are available here: [KaiyueWei/mongo-server-44991](https://github.com/KaiyueWei/mongo-server-44991).

If you want to reproduce the same analysis style, these are the core artifacts I generated:

*   `results/flamegraph_mongo-4.0.13.svg`
*   `results/flamegraph_mongo-4.2.1.svg`
*   `gdb.log` (stack traces + conditional breakpoint hits)
*   `repro/repro_perf_v3.sh` (workload + perf profiling automation)
