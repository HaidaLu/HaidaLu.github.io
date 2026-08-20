+++
date = '2026-08-20T16:54:17+02:00'
draft = false
title = 'RedisPersistent'
tags = ['distributed-systems']
+++
# Redis Persistence: RDB vs AOF by Actions

Redis is in-memory, but almost nobody runs it *only* in memory in production — a process restart, an OOM kill, or a bad deploy shouldn't mean the data is just gone. Redis offers two persistence mechanisms to guard against that: **RDB** (snapshotting) and **AOF** (append-only logging), and you can run either one alone or both together.

The docs explain the mechanics well enough, but reading "RDB can lose data between snapshots" is not the same as *watching it happen*. So instead of taking the trade-offs on faith, I ran three Redis instances side by side — RDB-only, AOF-only, and both — and put them through the same crash scenarios to see the differences directly.

This post walks through: how each mechanism actually works, the experiments I designed to expose their differences, and what the raw output showed.

## How RDB works

RDB persistence works by periodically forking the process and writing the entire in-memory dataset to a single binary file (`dump.rdb`) as a point-in-time snapshot.

Key config used in this experiment (`deploy/redis/redis-rdb/redis.conf`):

```
appendonly no
save 300 1        # if at least 1 key changed, snapshot after 300s
dbfilename dump.rdb
dir /data
```

The important detail that's easy to miss: `save 300 1` does **not** mean "300 seconds after each write." The 300-second clock starts from the last save (or process start), and Redis's background cron checks once a second whether that window has elapsed **and** at least one write happened since. Only then does it trigger a `BGSAVE`. That means:

- If the server just started and hasn't hit its first save window yet, a crash loses **everything** written so far — not just "the last few writes."
- On a clean shutdown (`SIGTERM`), Redis saves once regardless of the schedule, so planned restarts are safe even under RDB-only.

## How AOF works

AOF persistence works by logging every write command to an append-only file, and replaying that log on startup to rebuild the dataset.

Key config used (`deploy/redis/redis-aof/redis.conf`):

```
appendonly yes
appendfsync everysec   # fsync at most once per second
appenddirname "appendonlydir"
appendfilename "appendonly.aof"
save ""                 # RDB snapshotting disabled entirely
```

With `appendfsync everysec`, the worst-case loss window is one second of writes — far tighter than RDB's minutes-wide window. In Redis 7+, the AOF directory (`appendonlydir/`) actually contains a `manifest`, a `.base.rdb` file (the compacted starting point — itself stored in RDB format!), and one or more `.incr.aof` files holding the raw commands written since the last rewrite.

## The "both" configuration

`deploy/redis/redis-both/redis.conf` enables both:

```
appendonly yes
appendfsync everysec
save 60 1
```

On restart, when `appendonly` is `yes`, Redis **always prefers the AOF file** to reconstruct state — the RDB snapshot sitting next to it is not consulted for automatic recovery. This matters for how you interpret "both" mode's results below.

## The experiment setup

I built a repeatable test harness (`deploy/test/test.bash`) that runs three isolated containers — one per persistence mode — each with its own Docker volume so the experiments don't cross-contaminate:

```bash
docker run -d --name redis-dev-rdb  -p 6379:6379 -v redis-data-rdb:/data  -v .../redis-rdb/redis.conf:/usr/local/etc/redis/redis.conf  redis:7.4 redis-server /usr/local/etc/redis/redis.conf
docker run -d --name redis-dev-aof  -p 6380:6379 -v redis-data-aof:/data  -v .../redis-aof/redis.conf:/usr/local/etc/redis/redis.conf  redis:7.4 redis-server /usr/local/etc/redis/redis.conf
docker run -d --name redis-dev-both -p 6381:6379 -v redis-data-both:/data -v .../redis-both/redis.conf:/usr/local/etc/redis/redis.conf redis:7.4 redis-server /usr/local/etc/redis/redis.conf
```

Three things I set out to verify:

1. **Crash data loss** — write data, then `kill -9` the process (no chance to save). How much survives?
2. **Graceful shutdown** — write data, then `docker stop` (SIGTERM). Does the outcome differ from a hard kill?
3. **Recovery time** — with a large dataset, how much longer does replaying an AOF take compared to loading an RDB snapshot?

## Experiment 1: crash (`kill -9`)

Fresh containers, 200 seed keys written, then one more key (`marker`) written, followed immediately (within ~1s — well inside RDB's 300s window, so no snapshot had ever fired) by `docker kill -9` and a restart:

| mode | dbsize after restart | `marker` |
|---|---|---|
| **rdb** | **0** | **lost** |
| aof | 201 | recovered |
| both | 201 | recovered |

RDB didn't just lose the last write — it lost **the entire dataset**, because no snapshot had ever completed before the kill. This is the sharpest way to see RDB's risk: it's not a "lose the last few seconds" story, it's "lose everything since the last successful snapshot," which could be the whole dataset for a freshly (re)started instance.

AOF and "both" both fully recovered, because every write had already been fsynced to the append log within the previous second.

## Experiment 2: graceful shutdown (`docker stop`)

Same containers (RDB was still at 0 keys from the previous crash), write one key (`marker2`), then `docker stop` (SIGTERM) instead of `kill -9`:

| mode | dbsize after restart | `marker2` |
|---|---|---|
| **rdb** | **1** | **recovered** |
| aof | 202 | recovered |
| both | 202 | recovered |

This is the key contrast with Experiment 1: same "haven't hit the save window yet" situation, but this time RDB **did not lose the write**. A clean SIGTERM makes Redis save on its way out regardless of the configured schedule — so RDB's data-loss risk is specifically an *unclean crash* problem (OOM kill, power loss, `kill -9`), not a *restart* problem. Routine deploys and restarts are safe even with RDB alone; the danger is the crash you didn't plan for.

## Experiment 3: recovery time at scale

Flushed all three instances, wrote 100,000 simple keys via a single server-side `EVAL` script (to avoid round-trip overhead skewing the timing), then `docker restart` each container and read Redis's own self-reported load time from the logs:

| mode | log line | time |
|---|---|---|
| **rdb** | `DB loaded from disk` | **0.017s** |
| aof | `DB loaded from append only file` | 0.047s |
| both | `DB loaded from append only file` | 0.045s |

Two findings here:

- **RDB loaded ~2.7x faster.** Loading an RDB file is a straight binary deserialization of the final state; replaying an AOF means re-executing every logged command in order. For 100k simple `SET`s the absolute gap is small, but it scales with the number and complexity of operations — a workload with millions of `HSET`/`ZADD`/`RPUSH` calls against the same keys would show a much wider gap, since AOF has to redo all that history to arrive at the same end state that RDB stores directly.
- **"Both" recovered like AOF, not like RDB.** 0.045s is in the same ballpark as AOF's 0.047s, nowhere near RDB's 0.017s — confirming that enabling both doesn't make automatic recovery faster. The RDB snapshot sitting in the volume was simply not used for this restart.

## Summary

| Property | RDB only | AOF only | Both |
|---|---|---|---|
| Worst-case data loss (crash) | Everything since last snapshot (potentially all data) | ~1s (`everysec`) | ~1s (recovery goes through AOF) |
| Data loss on clean shutdown | None (saves on SIGTERM) | None | None |
| File format | Single binary snapshot | Directory of readable RESP commands + RDB base | Both files exist |
| Recovery speed (100k keys) | 0.017s | 0.047s | 0.045s (uses AOF, not the RDB file) |
| Disk growth pattern | Bounded by current dataset size | Grows with write volume until rewrite | Same AOF growth, plus a snapshot file |
| Good for | Fast recovery, portable backups, cheap clones | Crash safety | Crash safety (from AOF) + backup/migration convenience (from RDB) |

The practical takeaway: **RDB and AOF solve different problems, and "both" isn't redundant — it's two independent tools stacked.** AOF is what actually protects you from losing data on an unclean crash; RDB doesn't add extra loss protection on top of that (recovery ignores it), but it gives you fast recovery for cloning/migrating instances, a portable single-file backup format, and a second, structurally independent recovery point if the AOF itself ever gets corrupted. If your data matters — which for a storage engine, by definition, it does — AOF is close to a hard requirement, and RDB alongside it is cheap insurance, not overkill.

## Reproducing this

The full harness is at [`deploy/test/test.bash`](deploy/test/test.bash), with individual sub-commands (`reset`, `start`, `seed`, `crash`, `graceful`, `rto`, `status`) so any single experiment can be re-run on its own:

```bash
bash deploy/test/test.bash all   # reset + start + seed + crash + graceful
bash deploy/test/test.bash rto   # recovery-time comparison
```
