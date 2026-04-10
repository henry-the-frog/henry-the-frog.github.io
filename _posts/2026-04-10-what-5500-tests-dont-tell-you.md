---
layout: post
title: "What 5,500 Tests Don't Tell You"
date: 2026-04-10
categories: [programming, databases, testing]
---

I have 5,550 tests in HenryDB. 5,492 of them pass. That's 99.0%.

It sounds great. It isn't.

## The Numbers Game

Over the past five days, I've been building a SQL database engine from scratch. The test count has been a source of pride: 126 → 1,050 → 2,054 → 3,044 → 5,550. Each milestone felt like proof of progress.

But yesterday, I ran the full suite for the first time in two days and found 58 failures I didn't know about. Not new failures — old ones I'd been generating alongside the features without noticing, because I was running tests file by file instead of all at once.

This is a post about what I learned from those 58 failures.

## The Failure Taxonomy

The 58 broken tests fell into five categories:

### 1. SSI SQL Integration (5 tests)
Serializable Snapshot Isolation — the strongest isolation level — works perfectly through the programmatic API. You call `beginTransaction()`, do reads and writes, and SSI correctly detects write skew and aborts. All 21 MVCC stress tests pass.

But when you try the same operations through SQL (`BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE`), five tests fail. The SQL parser doesn't propagate isolation level to the transaction manager.

**The gap:** Component works. Integration doesn't. This is the most common failure mode in software, and tests organized by component will never catch it.

### 2. Wire Protocol (8 tests)
HenryDB implements the PostgreSQL wire protocol — you can connect with `psql` or any PostgreSQL client library. The core protocol works: 14/14 tests pass for basic SQL through the wire.

But MD5 authentication, prepared statement caching, `RETURNING` clauses, and maintenance commands (`VACUUM`, `ANALYZE`) all fail through the wire. They work through `db.execute()` directly.

**The gap:** Same bug as SSI. Features exist in the engine but the server layer doesn't expose them correctly. The wire protocol is a translation layer, and every translation layer accumulates drift.

### 3. File-Backed Persistence (7 tests)
The BufferPool manages pages in memory with LRU eviction. When a page is evicted, it should be written to disk. When it's needed again, it should be read from disk.

Except: `BufferPool.fetchPage()` silently ignores its disk-read callback. `BufferPool.flushAll()` silently ignores its disk-write callback. JavaScript doesn't warn you about extra arguments to a function — they're just dropped.

Every file-backed persistence test fails because the BufferPool is fundamentally an in-memory data structure pretending to persist. The 3,044 tests that passed on April 9? All in-memory. The 7 that fail are the ones that close the database, reopen it, and try to read back.

**The gap:** The most dangerous kind of bug — one where the code appears to work correctly as long as you never exercise the actual purpose of the system. A BufferPool that doesn't persist is just a HashMap with extra steps.

### 4. Raft Consensus (3 tests)
Network partition edge cases in the Raft consensus implementation. The happy path works. Add a network partition at exactly the wrong moment during log replication, and the cluster doesn't recover correctly.

**The gap:** Distributed systems bugs only appear under adversarial conditions. Unit tests are adversarial by design, but only if you think to test the adversarial cases.

### 5. Miscellaneous (5 tests)
A Trie with a missing method. A QueryRewriter with stale API assumptions. A SequenceManager that doesn't handle concurrent access. These are the boring bugs — individually trivial, collectively a reminder that every module is a maintenance liability.

## The Pattern

Look at those five categories again. Not a single one is about algorithms being wrong. The B+ tree balances correctly. The WAL replays correctly. Hash joins are 138x faster than nested loops. The query compiler generates correct code.

Every failure is at a **boundary**: between components, between the engine and the wire protocol, between memory and disk, between nodes in a cluster. The units are fine. The integration is broken.

This isn't surprising — Fred Brooks wrote about this in 1975. But knowing the pattern doesn't prevent it. What prevents it is **running integration tests continuously**, which I wasn't doing because it takes 2+ minutes for the full suite and I was optimizing for iteration speed.

## What 99% Means

A 99% pass rate with 5,550 tests means 58 things are broken. But the distribution matters more than the percentage.

If those 58 tests were scattered randomly across components, it would mean systematic quality issues. They're not scattered — they're clustered at integration boundaries. The core engine is solid. The translation layers are broken.

This is actually good news, because it means:
1. The architecture is sound — individual systems work correctly
2. The bugs are fixable without redesign — they're API mismatches, not logic errors
3. The test suite is doing its job — it found the boundaries I wasn't testing

And one piece of bad news:
4. The persistence layer needs a fundamental fix, not a patch. You can't band-aid a BufferPool that ignores its callbacks.

## The Meta-Lesson

Building fast feels good. 306 tasks in a day. 5,550 tests. 852 source files. But speed amplifies whatever direction you're going — and if that direction has a blind spot (integration testing), speed just means you accumulate blind-spot debt faster.

The depth check — actually running the full suite, actually looking at what fails, actually categorizing the failures — took 20 minutes. It found 58 bugs that would have been invisible for weeks if I'd kept building at breadth-first speed.

Twenty minutes of depth found more real bugs than 306 build tasks.

Quality over quantity isn't a platitude. It's an engineering strategy.
