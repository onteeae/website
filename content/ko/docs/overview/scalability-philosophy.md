---
title: 확장성 철학
weight: 3 
aliases: ['/docs/launching/scalability-philosophy/']
---

확장성 문제는 여러 가지 접근법을 통해 해결할 수 있다. 이 문서는 이러한 문제를 해결하기 위한 Vitess의 접근방식을 설명한다.

## 소규모 사례

데이터베이스를 더 작은 부분으로 분할하기로 결정할 때, 하나의 머신에 들어갈 정도로만 분할하는 것은 유혹적이다. 업계에선 호스트당 하나의 MySQL 인스턴스만 실행하는 것이 일반적이다.

그러나 Vitess는 인스턴스를 관리 가능한 부분(MySQL 서버당 250GB)으로 분할하고, 하나의 호스트에서 여러 인스턴스를 실행하는 것을 금기시하지 않도록 권장한다. 그렇게 해도 순수 하드웨어 자원의 사용량은 거의 같을 것이다. 그러나 MySQL 인스턴스가 작을 경우 관리의 용이성은 크게 향상된다. 각각 인스턴스의 포트들을 추적하고 경로를 분리하는 문제가 있다. 하지만, 이 장애물을 넘기면 다른 모든 것들은 더 단순해진다.

인스턴스 크기가 작을 때 잠금 경합이 적고, 복제가 훨씬 더 용이하며, 장애로 인한 서비스 영향이 줄어들고, 백업과 복원이 더 빨리 실행되며, 그 밖에 훨씬 더 많은 이차적인 이점을 얻을 수 있다. 예를 들어, 더 나은 서버 랙 다양성을 위해 인스턴스를 이리저리 옮길 수 있고, 이는 장애로 인한 서비스 영향과 자원 사용량을 더욱 줄일 수 있다.

## 복제를 통한 내구성

전통적인 데이터 스토리지 소프트웨어는 데이터를 디스크로 Flush하면 그것이 오래 유지되는 것으로 여긴다. 그러나 이러한 접근법은 오늘날의 하드웨어 세계에서는 실용적이지 않다. 그러한 접근법은 장애 시나리오에도 취약하다.

내구성에 대한 새로운 접근방식은 데이터를 여러 기계에, 심지어는 여러 지리적 위치에 복제함으로써 달성된다. 이러한 형태의 내구성은 기기 고장과 다른 장애에 대한 오늘날의 걱정을 해소한다.

Vitess의 많은 워크플로우는 이러한 접근방식을 염두에 두고 구축되었다. 예를 들어, 반동기 복제를 설정하는 것이 가장 권장된다. 이렇게 하면 데이터 손실 없이 마스터가 작동 중단될 때 Vitess가 새 복제본으로 페일오버할 수 있다. Vitess는 또한 당신이 손상된 데이터베이스를 복구하는 것을 피할 것을 권고한다. 대신 최근 백업에서 새로 생성하여 따라잡도록 두십시오.

복제를 사용하면 디스크 기반 내구성 설정 중 일부를 해제할 수도 있다. 예를 들어 sync_binlog를 끄면 디스크에 대한 IOPS 수가 크게 감소하여 유효 처리량을 높일 수 있다.


## Durability through replication

Traditional data storage software treated data as durable as soon as it was flushed to disk. However, this approach is impractical in today’s world of commodity hardware. Such an approach also does not address disaster scenarios.

The new approach to durability is achieved by copying the data to multiple machines, and even geographical locations. This form of durability addresses the modern concerns of device failures and disasters.

Many of the workflows in Vitess have been built with this approach in mind. For example, turning on semi-sync replication is highly recommended. This allows Vitess to failover to a new replica when a master goes down, with no data loss. Vitess also recommends that you avoid recovering a crashed database. Instead, create a fresh one from a recent backup and let it catch up.

Relying on replication also allows you to loosen some of the disk-based durability settings. For example, you can turn off `sync_binlog`, which greatly reduces the number of IOPS to the disk thereby increasing effective throughput.

## Consistency model

Before sharding or moving tables to different keyspaces, the application needs to be verified (or changed) such that it can tolerate the following changes:

* Cross-shard reads may not be consistent with each other. Conversely, the sharding decision should also attempt to minimize such occurrences because cross-shard reads are more expensive.
* In "best-effort mode", cross-shard transactions can fail in the middle and result in partial commits. You could instead use "2PC mode" transactions that give you distributed atomic guarantees. However, choosing this option increases the write cost by approximately 50%.

Single shard transactions continue to remain ACID, just like MySQL supports it.

If there are read-only code paths that can tolerate slightly stale data, the queries should be sent to REPLICA tablets for OLTP, and RDONLY tablets for OLAP workloads. This allows you to scale your read traffic more easily, and gives you the ability to distribute them geographically.

This trade-off allows for better throughput at the expense of stale or possibly inconsistent reads, since the reads may be lagging behind the master, as data changes (and possibly with varying lag on different shards). To mitigate this, VTGate servers are capable of monitoring replica lag and can be configured to avoid serving data from instances that are lagging beyond X seconds.

For a true snapshot, queries must be sent to the master within a transaction. For read-after-write consistency, reading from the master without a transaction is sufficient.

To summarize, these are the various levels of consistency supported:

* `REPLICA/RDONLY` read: Servers can be scaled geographically. Local reads are fast, but can be stale depending on replica lag.
* `MASTER` read: There is only one worldwide master per shard. Reads coming from remote locations will be subject to network latency and reliability, but the data will be up-to-date (read-after-write consistency). The isolation level is `READ_COMMITTED`.
* `MASTER` transactions: These exhibit the same properties as MASTER reads. However, you get REPEATABLE_READ consistency and ACID writes for a single shard. Support is underway for cross-shard Atomic transactions.

As for atomicity, the following levels are supported:

* `SINGLE`: disallow multi-db transactions.
* `MULTI`: multi-db transactions with best effort commit.
* `TWOPC`: multi-db transactions with 2PC commit.

### No multi-master

Vitess doesn’t support multi-master setup. It has alternate ways of addressing most of the use cases that are typically solved by multi-master:

* Scalability: There are situations where multi-master gives you a little bit of additional runway. However, since the statements have to eventually be applied to all masters, it’s not a sustainable strategy. Vitess addresses this problem through sharding, which can scale indefinitely.
* High availability: Vitess integrates with Orchestrator, which is capable of performing a failover to a new master within seconds of failure detection. This is usually sufficient for most applications.
* Low-latency geographically distributed writes: This is one case that is not addressed by Vitess. The current recommendation is to absorb the latency cost of long-distance round-trips for writes. If the data distribution allows, you still have the option of sharding based on geographic affinity. You can then setup masters for different shards to be in different geographic location. This way, most of the master writes can still be local.

## Multi-cell

Vitess is meant to run in multiple data centers / regions / cells. In this part, we'll use "cell" to mean a set of servers that are very close together, and share the same regional availability.

A cell typically contains a set of tablets, a vtgate pool, and app servers that use the Vitess cluster. With Vitess, all components can be configured and brought up as needed:

* The master for a shard can be in any cell. If cross-cell master access is required, vtgate can be configured to do so easily (by passing the cell that contains the master as a cell to watch).
* It is not uncommon to have the cells that can contain the master be more provisioned than read-only serving cells. These *master-capable* cells may need one more replica to handle a possible failover, while still maintaining the same replica serving capacity.
* Failing over from a master in one cell to a master in a different cell is no different than a local failover. It has an implication on traffic and latency, but if the application traffic also gets re-directed to the new cell, the end result is stable.
* It is also possible to have some shards with a master in one cell, and some other shards with their master in another cell. vtgate will just route the traffic to the right place, incurring extra latency cost only on the remote access. For instance, creating U.S. user records in a database with masters in the U.S. and European user records in a database with masters in Europe is easy to do. Replicas can exist in every cell anyway, and serve the replica traffic quickly.
* Replica serving cells are a good compromise to reduce user-visible latency: they only contain replica servers, and master access is always done remotely. If the application profile is mostly reads, this works really well.
* Not all cells need `rdonly` (or batch) instances. Only the cells that run batch jobs, or OLAP jobs, really need them.

Note Vitess uses local-cell data first, and is very resilient to any cell going down (most of our processes handle that case gracefully).
