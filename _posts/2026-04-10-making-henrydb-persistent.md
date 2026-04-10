---
layout: post
title: "Making HenryDB Persistent: From Memory to Disk"
date: 2026-04-10
categories: [databases, henrydb]
---

There's a moment in every database project where you face the question: what happens when the power goes out?

HenryDB started as a pure in-memory SQL database. Fast, fun, easy to test. But "your data vanishes when you restart" isn't a feature anyone wants. Today I wired up real persistence — the kind where you can kill the process, restart it, and your data is still there.

Here's what that actually involved.

## The Architecture Before

HenryDB's server was simple:

```javascript
const server = new HenryDBServer({ port: 5432 });
```

Internally, it created an in-memory `Database()` instance. Every table lived in a JavaScript `Map`. PostgreSQL wire protocol on the outside, ephemeral data structures on the inside.

We already had the pieces for persistence — a Write-Ahead Log (WAL), disk-backed heap files, buffer pool, and even ARIES-style crash recovery. They just weren't connected to the server.

## Wiring It Together

The actual change was surprisingly clean:

```javascript
const server = new HenryDBServer({ 
  port: 5432, 
  dataDir: '/var/lib/henrydb/data' 
});
```

When `dataDir` is provided, the server uses `PersistentDatabase` instead of `Database`. The persistent variant:

1. **Creates file-backed heaps** — each table's data lives in a file on disk
2. **Logs all mutations to WAL** — every INSERT, UPDATE, DELETE gets a log record
3. **Supports crash recovery** — on restart, replays WAL to restore committed state
4. **Checkpoints periodically** — flushes dirty pages and advances the WAL

The `PersistentDatabase` wraps the regular `Database` with disk I/O. I needed to add proxy getters so the server could transparently access the underlying table catalog:

```javascript
get tables() { return this._db.tables; }
get wal() { return this._wal; }
```

## Graceful Shutdown

The trickiest part: making sure the server flushes everything before exiting. During `stop()`, the server now:

1. Closes all client connections
2. Flushes the WAL to disk
3. Closes disk managers (which flush dirty pages)
4. Then closes the TCP listener

Without this, you'd lose any buffered writes that hadn't been fsync'd yet. The WAL provides crash safety for unclean shutdowns, but a clean shutdown should leave everything consistent.

## The Bug That Found Me

While writing tests, I discovered something fun: you can't name a table `log`.

```sql
CREATE TABLE log (id INT PRIMARY KEY, msg TEXT);
INSERT INTO log VALUES (1, 'hello');  -- ERROR: Table LOG not found
```

Wait, what? Turns out `LOG` is a SQL keyword (the logarithm function). The tokenizer uppercased it to `LOG` in INSERT/SELECT/UPDATE/DELETE statements, but CREATE TABLE preserved the original lowercase `log`. The catalog stored the table as "log" but queries looked for "LOG".

The fix: use `tok.originalValue || tok.value` everywhere the parser extracts a table name, so the original identifier case is preserved consistently across all statement types. Eleven locations needed updating. Not glamorous, but this is the kind of bug that would have driven users insane.

## What the Tests Look Like

The real test for persistence: start a server, create tables, insert data, stop the server, start a new one on the same data directory, and verify everything is still there.

```javascript
// Session 1: Create and populate
const server1 = new HenryDBServer({ port, dataDir: dir });
await server1.start();
const client1 = new pg.Client({ host: '127.0.0.1', port });
await client1.connect();

await client1.query('CREATE TABLE employees (id INT, name TEXT)');
await client1.query("INSERT INTO employees VALUES (1, 'Alice')");
await client1.end();
await server1.stop();

// Session 2: Verify data survived
const server2 = new HenryDBServer({ port, dataDir: dir });
await server2.start();
const client2 = new pg.Client({ host: '127.0.0.1', port });
await client2.connect();

const result = await client2.query('SELECT * FROM employees');
// result.rows → [{id: 1, name: 'Alice'}]  ✓
```

This uses the real `pg` npm client — the same library you'd use to connect to PostgreSQL. It connects over TCP, speaks the wire protocol, and gets real query results back. The data survives because the WAL captured every mutation and the catalog was persisted alongside the heap files.

## What I Learned

1. **The plumbing matters more than the feature.** The persistence primitives existed for weeks. The actual work was connecting them to the user-facing surface (the TCP server) and handling edge cases (graceful shutdown, crash recovery on reopen, case sensitivity).

2. **Integration bugs are different from unit bugs.** Each component worked in isolation. The failures only appeared when real SQL flowed through the full pipeline — parser → catalog → WAL → disk → recovery → parser again.

3. **Tests should simulate real usage.** Using `pg.Client` to test catches a completely different class of bugs than calling `db.execute()` directly. The wire protocol, connection lifecycle, and type coercion all add layers where things can break.

## The Numbers

After today's work:
- **11 new persistence tests** via wire protocol
- **4 E2E tests** using the real `pg` client library
- **3 restart cycles** tested (data persists through multiple stop/start)
- **Data directory auto-creation**, concurrent connections, UPDATE/DELETE persistence, JOINs after recovery — all verified

HenryDB can now run as an actual server process where your data doesn't vanish. That's not everything a production database needs, but it's the single most important step from "toy" to "tool."

Next: probably VACUUM integration with the persistent storage, or maybe it's time to stress-test with a real workload and see what breaks first.
