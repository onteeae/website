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

Vitess의 많은 작동 방식은 이러한 접근방식을 염두에 두고 구현되었다. 예를 들어, semi-sync Replication을 설정하는 것이 가장 권장된다. 이렇게 하면 마스터가 중단되도 데이터 손실없이 새 복제본으로 fail-over 가능하다. Vitess는 또한 당신이 손상된 데이터베이스를 복구하기 보다, 가장 최근 백업으로 새로이 구축할 것을 권고한다.

복제에 의존하면 디스크 기반 내구성 설정 중 일부를 해제할 수도 있다. 예를 들어 sync_binlog를 끄면 디스크에 대한 IOPS 수가 크게 감소하여 유효 처리량을 높일 수 있다.

## 일관성 모델

테이블을 샤딩하거나 다른 키 스페이스로 이동하기 전에 애플리케이션을 검증(또는 변경)하여 다음과 변경이 허용되도록 해야한다.

* 크로스-샤드 읽기는 서로 일관되지 않을 수 있다. 오히려 크로스 샤드 읽기는 더 비싸기 때문에 샤딩 결정은 그러한 발생(크로스-샤드 읽기)을 최소화하기 위해 시도해야 한다.
* "best-effort mode"에서는 크로스-샤드 트랜잭션이 중간에 실패하여 부분 커밋이 발생할 수 있다. 대신 분산 환경의 원자성을 보장해주는 "2PC(2-Phase-Commit) 모드" 트랜잭션도 대신 사용할 수 있다. 그러나 이 옵션을 선택하면 쓰기 비용이 약 50% 증가한다.

단일 샤드 트랜잭션은 MySQL이 지원하는 것처럼 계속 ACID를 보장한다.

만약 읽기 전용 코드에서 약간 오래된 데이터를 허용할 수 있다면, 쿼리를 OLTP를 위한 복제 테이블과 OLAP를 위한 읽기전용 테이블로 나누어 나누어 보낼 수 있다. 이렇게 하면 읽기 트래픽을 보다 쉽게 확장할 수 있으며, 지리적으로도 분산시킬 수 있다.

오래된 읽거나 일관되지 않을 수 있는 읽기를 일부 허용함으로써 더 나은 처리량을 제공한다. 그러한 비일관성은 데이터가 변경됨에 따라(그리고 다른 샤드의 지연으로) 마스터에 뒤처질 수 있기 때문에 발생할 수 있다. 이를 완화하기 위해 VTGate 서버는 복제 지연을 모니터링할 수 있으며 X초 이상 지연된 인스턴스의 데이터가 제공되지 않도록 구성할 수 있다.

For a true snapshot, queries must be sent to the master within a transaction. For read-after-write consistency, reading from the master without a transaction is sufficient.

실제 스냅샷을 보려면 트랜잭션 내에서 마스터에 쿼리해야 한다. 읽기 후 쓰기(read-after-write) 일관성을 위해서는 트랜잭션 없이 마스터에서 읽는 것으로 충분하다.

요약하면 다음과 같은 다양한 수준의 일관성이 지원된다.

* `REPLICA/RDONLY` read: Servers can be scaled geographically. Local reads are fast, but can be stale depending on replica lag.
* `MASTER` read: There is only one worldwide master per shard. Reads coming from remote locations will be subject to network latency and reliability, but the data will be up-to-date (read-after-write consistency). The isolation level is `READ_COMMITTED`.
* `MASTER` transactions: These exhibit the same properties as MASTER reads. However, you get REPEATABLE_READ consistency and ACID writes for a single shard. Support is underway for cross-shard Atomic transactions.

* "Replica/RDONLY" 읽기: 서버를 지리적으로 확장할 수 있다. 로컬 읽기는 빠르지만 복제본 지연에 따라 오래된 것일 수 있다.
* '마스터' 읽기: 세계적인 거장 한 명당 단 한 명뿐입니다. 원격 위치에서 전송되는 읽기는 네트워크 지연 시간 및 신뢰성의 영향을 받지만 데이터는 최신(읽기 후 쓰기 일관성)이 될 것이다. 격리 수준은 'READ'이다.커밋됨.
* '마스터' 트랜잭션: 이것들은 MASTER 읽기와 동일한 속성을 나타낸다. 단, 단일 샤드에 대해 REFITABLE_READ 일관성과 AID 쓰기를 얻을 수 있다. 크로스 샤드 아토믹 거래에 대한 지원이 진행 중이다.

원자성에 대해서는 다음과 같은 수준이 지원된다.

* '싱글': 멀티 db 트랜잭션 허용 안 함
* '멀티(Multi)': 최선의 노력을 다하는 멀티 db 트랜잭션 커밋.
* 'TWOPC' : 2PC 커밋을 이용한 멀티 db 트랜잭션


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
