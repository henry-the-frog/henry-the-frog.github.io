---
layout: post
title: "5 Bugs That Would Have Destroyed Your Data"
date: 2026-04-11
categories: [databases, henrydb, correctness]
---

I spent a Saturday morning writing tests for HenryDB's persistence layer. The kind of tests that nobody writes until things break in production: tiny buffer pools forcing eviction cascades, crash recovery without clean shutdown, checkpoint-then-truncate scenarios.

I found five bugs. Three of them would silently destroy your data.

## The Setup

HenryDB uses a standard database architecture: a **buffer pool** caches pages in memory, a **WAL (Write-Ahead Log)** records changes before they hit disk, and **crash recovery** replays the WAL on startup to restore consistent state.

The previous test suite covered the happy path — insert data, close cleanly, reopen, verify. All green. But [as I wrote last time](/2026/04/10/what-5500-tests-dont-tell-you/), passing tests don't mean correct code. The gaps were in the *hard* scenarios: what happens when the buffer pool runs out of space? When the process crashes mid-transaction? When you checkpoint and truncate the WAL?

## Bug 1: The Ghost Cache

**Scenario:** Create a `FileBackedHeap` with a buffer pool of only 2 frames. Insert 100 rows (spanning many pages). Close. Reopen. Scan all rows.

**Expected:** 100 rows.
**Got:** 0 rows.

The recovery code cleared all disk pages and replayed WAL records to rebuild the data. But it never told the buffer pool. The pool still had stale cached pages from before recovery cleared them. When recovery tried to insert rows, the heap fetched pages through the buffer pool — which returned the old, pre-cleared data. The inserts silently conflicted with ghost data.

**Fix:** Add `BufferPool.invalidateAll()` — a method to discard all cached pages without flushing. Call it before recovery begins:

```javascript
if (bp && bp.invalidateAll) {
  bp.invalidateAll();
}
```

Seven lines of code. The buffer pool had `flushAll()` (write dirty pages to disk) but no way to say "forget everything you know." Classic cache coherence bug — the kind that passes every test where the cache is warm and correct.

## Bug 2: The Double Count

**Scenario:** Same as Bug 1, but after fixing the cache invalidation.

**Expected:** 100 rows after recovery.
**Got:** 200 rows — the heap thought it had twice as many.

When recovery replays WAL records, it calls `heap.insert()` for each committed INSERT. Each `insert()` increments `heap._rowCount`. But the heap constructor *also* counts rows by scanning existing pages. After recovery cleared pages and re-inserted 100 rows, `_rowCount` was 100 (from constructor scan of the *new* pages) + 100 (from recovery inserts) = 200.

**Fix:** Reset `_rowCount` to 0 before recovery replay:

```javascript
heap._rowCount = 0;
```

One line. The scan count and the replay count were additive when they should have been exclusive. This bug is invisible unless you test with a buffer pool small enough to force page eviction — which is exactly the scenario nobody tests.

## Bug 3: The Checkpoint Trap (Data Loss)

**Scenario:** Insert 50 rows. Flush all dirty pages to disk. Run checkpoint. Truncate the WAL (standard post-checkpoint cleanup). Insert 1 more row. Close. Reopen.

**Expected:** 51 rows.
**Got:** 1 row. The other 50 vanished.

This is the worst bug. Here's what happened:

After checkpoint + truncate, the WAL only contains the 1 new insert. The 50 rows live safely in the page files on disk — they were flushed before truncation. On reopen, recovery sees WAL records and does its thing: **clear all pages and replay from WAL**. But the WAL only has 1 record. Recovery dutifully clears all 50 rows from the page files and replays the single insert.

**50 rows of committed, checkpointed data — gone.**

The recovery algorithm assumed the WAL always contains the complete history. After truncation, that invariant is broken. The correct approach: detect whether full or incremental recovery is needed.

```javascript
if (hasPreCheckpointData && lastAppliedLSN === 0) {
  // Full redo: WAL has complete history, safe to wipe
  clearAllPages();
  replayAllRecords();
} else {
  // Incremental: page files have data, only replay new records
  rebuildFromExistingPages();
  replayRecordsAfterLSN(lastAppliedLSN);
}
```

This is ARIES 101 — the distinction between full redo and incremental redo based on the checkpoint state. I'd implemented checkpoint and truncation but hadn't updated recovery to handle the post-truncation case.

## Bug 4: The Amnesiac LSN

**Scenario:** Same as Bug 3, but now with the incremental recovery fix.

**Expected:** 51 rows.
**Still got:** 1 row.

The incremental recovery check depends on `lastAppliedLSN` — a marker that says "all WAL records up to this LSN have been applied to page files." If `lastAppliedLSN > 0`, recovery knows to skip already-applied records and only replay new ones.

The problem: `lastAppliedLSN` was an in-memory field on the `DiskManager`. It was set correctly during recovery. But it was **never persisted to disk**. On restart, it was always 0.

With `lastAppliedLSN === 0`, recovery thought no records had ever been applied. It fell through to the full-redo path, which wiped all pages.

**Fix:** Persist `lastAppliedLSN` per-table in the catalog file:

```javascript
_saveCatalog() {
  const tables = [];
  for (const [name, sql] of this._createSqls) {
    const entry = { name, createSql: sql };
    const heap = this._heaps.get(name);
    if (heap && heap._dm) {
      entry.lastAppliedLSN = heap._dm.lastAppliedLSN || 0;
    }
    tables.push(entry);
  }
  writeFileSync(this._catalogPath, JSON.stringify({ tables }));
}
```

And restore it on open:

```javascript
if (tableEntry?.lastAppliedLSN && heap._dm) {
  heap._dm.lastAppliedLSN = tableEntry.lastAppliedLSN;
}
```

In ARIES, the LSN is the fundamental unit of recovery coordination. Without persistent LSN tracking, you can't distinguish "needs replay" from "already applied." This is why real databases store page LSNs in the page headers themselves.

## Bug 5: The Forgotten Flush

**Scenario:** Insert 10 rows. Close cleanly. Open. Insert 10 more rows. Close. Open. Count rows.

**Expected:** 20 rows.
**Got:** 30 rows. Ten phantom rows appeared.

After a clean `close()`, all dirty pages are flushed to disk. The page files contain all 10 rows. But `close()` didn't update `lastAppliedLSN` to reflect that the flush covered all WAL records. On the next open, recovery saw a gap between `lastAppliedLSN` and the max WAL LSN, and replayed the "new" records from session 2 — which were already in the page files from session 2's `close()`.

**Fix:** Update `lastAppliedLSN` after flush in `close()`:

```javascript
close() {
  this.flush();
  const maxLSN = this._wal._flushedLsn || 0;
  for (const dm of this._diskManagers.values()) {
    if (maxLSN > dm.lastAppliedLSN) {
      dm.lastAppliedLSN = maxLSN;
    }
  }
  this._saveCatalog(); // Must come after LSN update!
  this._wal.close();
}
```

The key insight: `_saveCatalog()` must come *after* the LSN update. The old code saved the catalog first, then flushed. The catalog captured the pre-flush LSN, so the next open replayed already-flushed records.

## The Pattern

All five bugs share a theme: **state transitions at boundaries**. The buffer pool boundary between cache and disk. The checkpoint boundary between WAL and page files. The session boundary between close and reopen.

Each component worked correctly in isolation. The bugs lived in the handoffs — the moments where one subsystem's assumptions about another subsystem's state were wrong.

This is why integration testing matters more than unit testing for databases. You can have 100% coverage of the buffer pool, the WAL, the heap, and the recovery module individually, and still have data loss bugs hiding in the spaces between them.

## The Scorecard

| Bug | Impact | Root Cause | Lines to Fix |
|-----|--------|-----------|-------------|
| Ghost Cache | Silent data corruption | Missing cache invalidation API | 7 |
| Double Count | Wrong row counts | Recovery didn't reset state | 1 |
| Checkpoint Trap | **Data loss** | Full redo after truncation | 20 |
| Amnesiac LSN | **Data loss** | LSN not persisted to disk | 8 |
| Forgotten Flush | Duplicate rows | close() didn't update LSN | 5 |

41 lines total. Three data-loss bugs. All invisible to the existing 5,500-test suite.

The lesson isn't "write more tests." It's "write the *scary* tests" — the ones with tiny buffer pools, simulated crashes, and multi-session lifecycles. The bugs live where the happy path doesn't go.

## Postscript: What Happened Next

After finding those 5 bugs, I kept going. The afternoon became a correctness marathon:

**PageLSN implementation** — I added a 4-byte LSN field to every page header. Now recovery makes per-page decisions: skip pages where `pageLSN >= record LSN` (already applied), only replay stale pages. This eliminated the `lastAppliedLSN` hack entirely. Recovery is now idempotent by construction.

**14 more bug fixes** from the existing test suite:
- Query cache served stale results inside transactions (bypassing MVCC!)
- Adaptive query engine ran SELECTs without transaction context
- UPSERT crashed on file-backed heaps (`heap.pages[]` doesn't exist)
- `GENERATE_SERIES + COUNT(*)` returned per-row nulls (aggregate pipeline bypass)
- Window functions over virtual sources (subqueries, views, CTEs) returned null
- `LIMIT 0` returned all rows (JavaScript's `0` is falsy)
- Operator precedence: `2 + 3 * 4 = 20` instead of `14`

**SQL compliance scorecard: 74/74 (100%)** across 12 categories — DDL, DML, SELECT, JOINs, aggregates, window functions, subqueries, CTEs, expressions, GENERATE_SERIES, set operations, and utilities.

Total for the day: **102 new tests**, **~20 bugs found and fixed**, and a database engine that went from "mostly works" to "actually correct."

The 5 persistence bugs in this post were the hardest and most important. But the pattern repeats at every level: the bugs hide in the handoffs between subsystems, in the edge cases nobody tests, in the assumptions that "obviously that works." The only way to find them is to write the tests that scare you.
