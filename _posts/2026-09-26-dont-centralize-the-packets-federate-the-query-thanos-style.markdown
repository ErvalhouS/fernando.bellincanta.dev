---
layout: post
title:  "Don't centralize the packets, federate the query: Thanos-style SIP capture"
description: "Keep SIP capture in the region that produced it and ship only query results across the network: the pattern, the sink it needed, and the traps."
date:   2026-09-26 09:00:00 -0300
categories: Concepts
permalink: /:categories/:title
---

The call ladder I needed was 41 rows. To get it, the architecture on the whiteboard wanted me to move every SIP packet from every region to one central box, forever.

That is the default shape of packet capture on most voice platforms: every proxy and media node mirrors HEP to a collector, the collector writes to one big database, and the troubleshooting UI queries it. It works. It is also the most expensive way I know to answer a question you ask a few dozen times a day about a few dozen rows.

I just finished bringing a new region into a capture setup that works the other way around. Each region keeps its own packets, and a central node that holds nothing fans the query out when someone asks for a Call-ID. If you have run Thanos Query over sidecars, you know the pattern. Here it is applied to signalling, with the sink it needed and the traps it came with.

## 1. Storage at the edge, a stateless reader in the middle

Every region runs a HEP receiver built on heplify-server and a local ClickHouse, with retention enforced by a local TTL. Nothing about capture leaves the region. The central node is a ClickHouse with no data, which knows the regions only as shards:

```xml
<remote_servers>
  <capture_fed>
    <shard>
      <replica>
        <host>edge-01.region-a.internal</host>
        <port>9440</port>
        <secure>1</secure>
        <user>default</user>
        <password from_env="FED_PW_REGION_A"/>
      </replica>
    </shard>
    <!-- one <shard> per region, rendered from one list in the values -->
  </capture_fed>
</remote_servers>
<openSSL><client>
  <caConfig>/etc/clickhouse-server/ca/ca.pem</caConfig>
  <verificationMode>strict</verificationMode>
</client></openSSL>
```

TLS on 9440 is verified against a private CA, and each region gets its own secret. On top sit `Distributed` tables, which store nothing: each shard runs the query against its own local table and only results come back. The lookup is an ordinary query:

```sql
SELECT create_date, region, src_ip, dst_ip, method
FROM hep.sip_all
WHERE match(sid, '${callid}')
  AND create_date > now() - INTERVAL 2 HOUR
ORDER BY create_date
LIMIT 500;
```

It is a regex rather than equality on purpose, because people paste partial Call-IDs. A "deep" toggle also searches the raw SIP payload for things like a phone number, and runs `UNION ALL` of both lookups. Useful, and also where the cost hides: the payload search is a full scan of the window on every region at once, so it stays opt-in. The time window is never optional, not even for the panels that only fill dropdowns, and the `LIMIT` caps what each region sends back when a search is too broad.

## 2. The sink nobody ships

The stock heplify-server has SQL drivers for MySQL and Postgres, and nothing else. Point it at ClickHouse and it logs `invalid DBDriver` and **keeps running**: the listener stays up, `/metrics` keeps counting, nothing is persisted. And unless you set `LogStd = true`, that one line goes to a file inside the container, not to `kubectl logs`. If you inherit a capture deployment, grep for it.

We tried the Loki output with a Loki-compatible layer on ClickHouse first. Every packet landed, but the HEP addresses exist only as labels: labels off, no source or destination IP; labels on, one series per message. So we wrote a ClickHouse sink next to the existing drivers, and the discipline mattered more than the code:

- **Minimal surface.** Idempotent setup (`IF NOT EXISTS`, retention as a table TTL), batched inserts with server-side `async_insert` and `wait_for_async_insert = 1`, bounded retries, then a loud drop and a metric. Retrying forever just moves the loss to the UDP socket.
- **Flat columns for the `WHERE` clause.** Call-ID, timestamp, addresses, SIP event, region. Everything else goes in a small JSON column.
- **Call-ID as the sort key.** On a synthetic 50-million-row table, `LIKE` over the raw payload scanned everything in about 30 seconds. A Call-ID column in the `ORDER BY` turns lookups into a primary-key range read.
- **Mirror the old schema.** The tables reproduce the layout of the collector we were replacing, the one the dashboards already query, and CI checks that on every build. The dashboards didn't change. Just never feed both writers the same HEP stream: nothing dedupes it.

That CI check also matters for federation. `IF NOT EXISTS` never alters a table, so a drifted shard stays drifted. The check against one reference layout is what keeps every shard identical.

## 3. A stateless node forgets its schema

The central node has no volume on purpose. But `Distributed` tables are DDL, and DDL lives on disk, so a rescheduled pod comes back as a healthy ClickHouse with zero tables.

So the schema ships with the deployment. The official image runs `/docker-entrypoint-initdb.d/` on an empty data directory, and without a volume that is every start. Three rules follow. `IF NOT EXISTS` everywhere. Never run `CREATE TABLE` by hand on that node, because it disappears at the next restart. And put a checksum of the *rendered* ConfigMap in the pod annotations, since `subPath` mounts never refresh and a schema change should roll the pod.

One caution about adding tables later: a `Distributed` table whose local table is missing on one shard fails the whole query. Create the local tables first. If a shard will never get one (a legacy store you don't own), define a second cluster without it.

## 4. One old shard, one misleading error

One region, the legacy store, still ran a ClickHouse from before the new analyzer became the default. Every federated query touching it failed like this:

```
DB::Exception: Missing columns: '__table1.sid' while processing query
```

The column existed. The initiator's analyzer rewrote the query with `__table1` aliases the old shard couldn't parse, so the error blamed the schema and hid the version skew. Upgrading a store you are about to retire is a poor use of a weekend, so we turned the analyzer off on the initiator, in the server profile rather than in each panel:

```xml
<profiles><default>
  <enable_analyzer>0</enable_analyzer> <!-- allow_experimental_analyzer before 24.8 -->
</default></profiles>
```

A per-query setting is one more thing each new panel has to remember. If I were adding a step to region onboarding, it would be comparing `version()` across all shards before trusting any "missing columns" error. We don't run that check yet.

## 5. Knowing where a row came from

The sink stamps SIP rows with a region, but legacy rows carry an empty one. We label those at read time instead of backfilling a table we don't own: `if(region = '', 'legacy', region)` in the filters and in the dropdown, so "legacy" is just another region.

The RTCP tables have no region at all. The ladder never notices, since it fetches RTCP by Call-ID alongside SIP rows. Per-region aggregates would. One idea I haven't built: `Distributed` exposes a virtual `_shard_num`, so a small mapping table in `init.sql` could label rows for free. Aggregate per shard in a subquery and join on the central node, because a direct join gets pushed to shards that don't have the table. Render the mapping from the same region list as `remote_servers`, since `_shard_num` is positional.

## 6. The cost is the network, not the disk

Capture is a firehose with a tiny read rate. Centralizing it means paying cross-region transfer on all of it so you can read well under 1% of it. Federated, only result sets cross a boundary, so traffic scales with how often humans ask questions, not with call volume. And the packets carrying phone numbers never leave the region they were captured in, which is starting to matter as much as the bill.

The trade-offs are real. A fan-out is as slow as the slowest region, and by default one region down fails the whole query. ClickHouse can return partial results with `skip_unavailable_shards = 1`. If you go that way, the UI has to say "3 of 4 regions answered", not pretend the fourth had no calls. We haven't turned it on.

## 7. Auth that proves something

Today each shard authenticates the central node as its default user, with a distinct secret per region. The read-only `reader` account lives on the central node, for the dashboards. That is decent, and it is also what I'd tighten next.

The common failure with setups like this is testing auth with a wrong password. That passes even on a user with an empty password, because a non-empty string still isn't the empty string. The test that proves something is trying the credential that should be rejected:

```bash
clickhouse-client -h "$h" --password '' -q 'SELECT 1' && echo "FAIL $h"
```

The other recommendation: the central node is the one component that can reach every region, so what it holds on each shard should be `SELECT` only. If it is compromised, it should leak capture data, not be able to delete it.

## What to actually take away

**Move the question to the data, not the data to the question.**

Keep a receiver and a local ClickHouse per region and a stateless node holding only `Distributed` tables. If your receiver has no ClickHouse sink, check it isn't silently dropping everything, and if you write one, mirror the old collector's schema and verify it in CI. Always query with a time window and a `LIMIT`, and keep payload search opt-in. Ship the central schema in an `init.sql` with no volume and checksum it into the pod. Fix analyzer skew in the server profile, not per panel. Label legacy rows at read time. And test auth with the credential that should fail.
