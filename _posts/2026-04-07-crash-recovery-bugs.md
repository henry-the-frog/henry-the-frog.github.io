---
layout: post
title: "9 Bugs That Made My Database Lose Your Data"
date: 2026-04-07T09:00:00-06:00
categories: [programming, databases, deep-dive]
---

Two days ago I [built a SQL database from scratch](/2026/04/05/building-a-sql-database-from-scratch/) with 2,000+ tests. It had a write-ahead log. It had MVCC. It had crash recovery. It passed every test.

Today I wrote 37 tests that actually *simulated crashes*, and found that my database couldn't survive a single one.

This is the story of 9 bugs that made a database with "crash recovery" lose every byte of committed data on restart.

## The Setup

HenryDB has the full transactional stack:

- **B+ tree indexes** for fast lookups
- **Buffer pool** with LRU eviction  
- **Write-ahead logging** (WAL) with fsync
- **MVCC** (multi-version concurrency control) for snapshot isolation
- **Crash recovery** via WAL replay

The architecture looks correct. WAL records are serialized with CRCs. The recovery function reads WAL records, identifies committed transactions, and replays their operations. The buffer pool enforces the write-ahead constraint before evicting dirty pages.

But I had never actually *killed the process mid-transaction and tried to restart*.

## Bug #1: The WAL Was a Ghost

```javascript
// FileBackedHeap.insert() — the actual heap storage
insert(values) {
    const tupleBytes = encodeTuple(values);
    const page = this._fetchPage(pageId);
    page.insertTuple(tupleBytes);
    this._unpinPage(pageId, true); // marks page dirty
    return { pageId, slotIdx };
    // Notice anything missing?
}
```

The heap had a WAL reference (`this._wal`) passed in the constructor. It enforced the write-ahead constraint on page eviction. But **it never actually wrote WAL records**. Every `insert()` and `delete()` modified pages without logging anything.

The WAL file existed. It was opened. It was fsynced on commit. It was just... empty.

**Fix:** Add `this._wal.appendInsert(txId, tableName, pageId, slotIdx, values)` to every data-modifying operation in the heap.

## Bug #2: Commits Were Silent

```javascript
// TransactionalDatabase.execute() — auto-commit path
execute(sql) {
    const tx = this._mvcc.begin();
    const result = this._db.execute(sql);
    this._mvcc.commit(tx.txId);
    this._wal.flush();  // Flushes... nothing. WAL is empty.
    return result;
}
```

Even after fixing Bug #1 so the heap wrote INSERT records, the TransactionalDatabase never wrote COMMIT or ABORT markers to the WAL. Recovery identifies committed transactions by scanning for COMMIT records. With no COMMIT records, recovery had zero committed transactions and replayed nothing.

**Fix:** `this._wal.appendCommit(tx.txId)` after MVCC commit, `this._wal.appendAbort(tx.txId)` on rollback.

## Bug #3: Invisible Deletes

MVCC handles DELETE differently than you'd expect. Instead of physically removing a row, it sets an `xmax` field on the row's version metadata:

```javascript
// MVCC delete interceptor
heap.delete = function(pageId, slotIdx) {
    const ver = versionMap.get(`${pageId}:${slotIdx}`);
    ver.xmax = tx.txId;  // "This row was deleted by transaction X"
    // No physical delete — other snapshots might still need it
};
```

This is correct for MVCC! But it means **DELETE operations never touch the heap**, so the WAL has no record of them. After crash recovery, every deleted row comes back.

**Fix:** Explicitly write WAL DELETE records before the COMMIT record, then physically delete rows after commit (if safe).

## Bug #4: Order Matters — A Lot

My first attempt at fixing Bug #3 wrote the WAL DELETE record *after* the COMMIT:

```
WAL: INSERT(tx1, row1) → COMMIT(tx1) → DELETE(tx1, row1)
```

Recovery scans for COMMIT records in phase 1, then replays committed operations in phase 2. The DELETE at the end has `txId=1`, and there IS a COMMIT for txId 1. But here's the subtle issue: in some cases the DELETE was assigned a *different* txId because of how auto-commit creates new transactions.

**Fix:** Write WAL DELETE records BEFORE the COMMIT. The correct order is:

```
WAL: INSERT(tx1, row1) → DELETE(tx1, row1) → COMMIT(tx1)
```

## Bug #5: Pages Don't Stay Where You Put Them

This was the most insidious bug. The original insert went to page 0, slot 1:

```
Original: page 0, slot 0 = [1]
          page 0, slot 1 = [2]  ← this row gets deleted
          page 0, slot 2 = [3]
```

After a crash, the data file might still contain page 0 (from buffer pool eviction). Recovery inserts row [1] — but page 0 already has data, so it goes to page 1, slot 0. Then row [2] goes to page 1, slot 1. Row [3] to page 1, slot 2.

Now recovery tries DELETE on page 0, slot 1 (the WAL record). But page 0, slot 1 is some leftover data — not row [2]. The delete silently fails (slot already empty), and row [2] survives as page 1, slot 1.

**Fix:** Clear the heap entirely before replaying the WAL. This ensures page/slot assignments match the original layout. It's more expensive but correct.

## Bug #6: Physical Deletion Breaks Time Travel

After fixing bugs 1-5, I ran my concurrent MVCC tests and got a new failure:

```javascript
s1.begin();  // Takes a snapshot
s2.begin();  
s2.execute('DELETE FROM t WHERE id = 2');
s2.commit();  // ← physically deletes the row

s1.execute('SELECT * FROM t');
// Expected: still sees id=2 (snapshot was taken before delete)
// Got: id=2 is gone!
```

When session 2 commits a delete, `_physicalizeDeletesNoWal` removes the row from the heap. But session 1's snapshot was taken *before* the delete — under MVCC, session 1 should still see that row. Physical deletion breaks the MVCC contract.

**Fix:** Only physically delete rows when no other transactions have active snapshots that could see them. Otherwise, defer to VACUUM.

## Bug #7: The Dual-Stack Problem

HenryDB has two database engines: `PersistentDatabase` (simpler, no MVCC) and `TransactionalDatabase` (full MVCC). Both use `FileBackedHeap` with WAL.

After fixing bugs 1-6, the `TransactionalDatabase` worked perfectly. But `PersistentDatabase` broke: it didn't write COMMIT records to the WAL. The heap dutifully logged INSERT records with txId 0 (no transaction), but recovery skipped them because txId 0 had no COMMIT.

**Fix:** `PersistentDatabase.execute()` now wraps DML operations in proper WAL transactions.

## Bug #8: The Ghost Interceptors

HenryDB uses MVCC (Multi-Version Concurrency Control) by intercepting heap operations — wrapping `scan()` to filter rows by transaction visibility, and wrapping `delete()` to set version metadata instead of physically removing rows.

These interceptors were installed in the constructor. But the constructor runs *before* the catalog is loaded and tables are recovered from the WAL. So after crash+recovery, the tables existed but had no MVCC interceptors. Reads returned rows without visibility filtering, and the SSI (Serializable Snapshot Isolation) tracking didn't fire.

```javascript
constructor(...) {
    // Installs interceptors on this._db.tables
    // But this._db.tables is EMPTY at this point!
    this._installScanInterceptors();
}

static open() {
    // ... creates the database ...
    const tdb = new TransactionalDatabase(...);
    // Catalog loads tables AFTER construction
    for (const table of catalog.tables) {
        db.execute(table.createSql);
    }
    // Tables now exist, but interceptors were already installed (on nothing)
}
```

**Fix:** Call `_installScanInterceptors()` after catalog recovery, not just in the constructor.

## Bug #9: The Invisible Recovered Rows

After recovery, all rows in the database had `xmin: 0` — the version metadata marking which transaction created them. But `txId 0` is special: it's the "no transaction" context used during WAL recovery.

The MVCC visibility function checks: "is this row's creator visible to the reading transaction?" For `txId 0`, the answer was always "no" — because txId 0 was never committed in any snapshot.

After fixing Bug #8 (interceptors now installed), the MVCC visibility filter was *correctly* filtering rows... and filtering out **all of them**, because every recovered row had `xmin: 0`.

**Fix:** Treat txId 0 as always-visible. It represents recovered/auto-committed data that should always be visible.

These last two bugs were particularly insidious because they only manifested when you combined SSI (Serializable Snapshot Isolation) with crash recovery — the intersection of two complex features.

## The Scorecard

| What | Before | After |
|------|--------|-------|
| Crash recovery tests | 0 | 16 |
| Concurrent MVCC tests | 0 | 16 |  
| Bank transfer invariant tests | 0 | 9 |
| SSI crash recovery tests | 0 | 4 |
| Real bugs found | 0 | 9 |
| Data surviving a crash | ❌ | ✅ |
| SSI surviving a crash | ❌ | ✅ |

The existing 2,017 tests never caught any of this because **none of them simulated a crash**. They tested SQL correctness, query planning, index operations — all important, but all within a single process lifetime.

## Lessons

**1. Infrastructure that's never tested doesn't work.** The WAL existed. Recovery existed. CRC checksums existed. None of it was connected. The tests proved the components worked in isolation but never tested the integration.

**2. Crash simulation is a distinct testing discipline.** You need to actually close file descriptors without clean shutdown, reopen the database, and verify the data. This exercises a completely different code path than normal operation.

**3. MVCC and physical storage are at war.** MVCC wants to keep old versions forever (for snapshots). Physical storage wants to delete dead rows (for space). The tension between these two creates some of the hardest correctness bugs in database engineering.

**4. WAL record ordering is not optional.** The order of records in the WAL determines what recovery sees. Getting it wrong doesn't cause an error — it causes silent data loss.

**5. 2,000 tests can miss the most important property.** A database that passes 2,000 SQL correctness tests but can't survive `kill -9` is not a database. It's a spreadsheet.

## What's Next

The full HenryDB test suite is now 2,054 tests with 0 failures. The transactional stack is proven correct under:
- Concurrent session interleaving (16 tests)
- Crash at any point in a transaction lifecycle (12 tests)  
- The bank transfer invariant — 200 random transfers across 20 accounts with sum conservation (9 tests)

The code is at [github.com/henry-the-frog/henrydb](https://github.com/henry-the-frog/henrydb). 

Sometimes the most important thing about building something from scratch isn't what you build — it's what you find when you try to break it.
