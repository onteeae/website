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

전통적인 데이터 스토리지 소프트웨어는 데이터를 디스크로 Flush하면 그것이 아주 오래 유지되는 것으로 여긴다. 그러나 이러한 접근법은 오늘날의 하드웨어 세계에서는 실용적이지 않다. 그러한 접근법은 장애 시나리오에도 취약하다.

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

실제 스냅샷 데이터를 보려면 트랜잭션 내에서 마스터에 쿼리해야 한다. 읽기 후 쓰기(read-after-write) 일관성을 위해서는 트랜잭션 없이 마스터에서 읽는 것으로 충분하다.

요약하면 다음과 같은 다양한 수준의 일관성이 지원된다.

* Replica/RDONLY Read: 서버를 지리적으로 확장할 수 있다. 로컬 읽기는 빠르지만 복제지연 따라 오래된 데이터일 수 있다.

* Master Read: 샤드당 단 하나의 Master만 존재한다. 지리적으로 먼 위치에서의 읽기는 지연시간 과 네트워크 신뢰성의 영향을 받지만 데이터는 최신(읽기 후 쓰기 일관성)이 될 것이다. 격리 수준은 READ_COMMITTED이다.

* Master Transaction: 외부적으로 MASTER 읽기와 동일한 속성을 가진다. 단, 단일 샤드에 대해 REFITABLE_READ 일관성과 ACID 쓰기를 보장 받을 수 있다. 크로스-샤드 원자성(Atomicity)를 위한 지원이 개발중이다.

원자성에 대해서는 다음과 같은 수준이 지원된다.

* SINGLE : 멀티-DB 트랜잭션을 허용하지 않음
* MULTI : Best-Effort 1PC 방식의 멀티 db 트랜잭션 커밋.
* TWOPC : 2PC 커밋 방식의 멀티 db 트랜잭션

### 멀티 마스터는 없음

Vitess는 멀티 마스터 설정을 지원하지 않는다. 그러나 Vitess는 멀티 마스터 설정이 해결하는 대부분의 use-case를 다루는 다른 방법을 가진다.

* 확장가능성: 멀티 마스터가 당신에게 약간의 추가적인 이점을 주는 상황이 있다. 하지만, 쿼리들이 결국은 모든 마스터들에게 적용되어야 하기 때문에, 그것은 지속 가능한 전략이 아니다. Vitess는 무한히 확장 가능한 샤딩을 통해 이 문제를 해결한다.
* 고가용성: Vitess는 오케스트레이터와 통합되어 장애 감지 후 몇 초 이내에 새 마스터로의 페일오버를 수행할 수 있다. 이것은 대부분의 응용엔 충분하다.
* 지연시간이 짧은, 지리적으로 분산된 쓰기: 이것은 Vitess 다루지 않은 또 하나의 사례이다. 현재 추천하는 방식은 장거리로 인한 네트워크 지연을 줄이는 것이다. 만약 데이터 분포가 이를 허용한다면, 지리적 인접도에 따른 샤딩 옵션을 사용할 수 있다. 그런 다음 다른 샤드의 마스터가 다른 지리적 위치에 있도록 설정할 수 있다. 이런 식으로 하면, 마스터 쓰기의 대부분은 여전히 지역적일 수 있다.


## 멀티 셀(Multi-Cell)

Vitess는 여러 데이터 센터/지역/셀에서 실행하도록 만들어졌다. 이 부분에서는 가까워서 서로 같은 지역적 가용성을 공유하는 서버들의 집합을 가리켜 "셀"이라는 용어로 부를 것이다.

셀에는 일반적으로 Vitess 클러스터를 사용하는 테이블, vtgate, 그리고 이를 사용하는 응용 서버들이 포함되어 있다. Vitess를 사용하면 모든 구성 요소를 필요에 따라 구성하고 가져올 수 있다.

* 샤드의 마스터는 어떤 셀에도 있을 수 있다. 마스터로의 크로스-셀 접근이 필요한 경우 (마스터가 들어 있는 셀을 넘겨주도록) 쉽게 vtgate를 설정할 수 있다.
* 마스터를 포함할 수 있는 셀을 읽기 전용 셀보다 더 많이 프로비저닝하는 것은 흔한 일이다. 이러한 *마스터를 가질 수 있는* 셀은 일어날 수 있는 페일오버를 대비하면서도 동일한 레플리카 가용량을 유지하기 위해 하나의 레플리카가 더 필요할 수 있다.
* 한 셀의 마스터에서 다른 셀의 마스터로 페일오버하는 것은 로컬 페일오버와 다르지 않다. 이는 트래픽과 응답 시간에 영향을 미치지만 애플리케이션 트래픽 또한 새로운 셀로 다시 방향을 바꾸면 최종 결과는 안정적이다.
* 한 세포에 주인의 파편이 있는 파편이 있고, 다른 세포에 주인의 파편이 있는 파편이 있는 것도 가능하다. vtgate는 원격 액세스에서만 추가 지연 비용을 발생시켜 트래픽을 올바른 위치로 라우팅할 뿐이다. 예를 들어 미국 마스터가 있는 데이터베이스에 미국 사용자 레코드를 만들고 유럽 마스터가 있는 데이터베이스에 유럽 사용자 레코드를 만드는 것은 쉬운 일이다. 복제본은 모든 셀에 존재할 수 있으며, 복제본 트래픽을 신속하게 처리할 수 있다.

* It is also possible to have some shards with a master in one cell, and some other shards with their master in another cell. vtgate will just route the traffic to the right place, incurring extra latency cost only on the remote access. For instance, creating U.S. user records in a database with masters in the U.S. and European user records in a database with masters in Europe is easy to do. Replicas can exist in every cell anyway, and serve the replica traffic quickly.
* Replica serving cells are a good compromise to reduce user-visible latency: they only contain replica servers, and master access is always done remotely. If the application profile is mostly reads, this works really well.
* Not all cells need `rdonly` (or batch) instances. Only the cells that run batch jobs, or OLAP jobs, really need them.

Note Vitess uses local-cell data first, and is very resilient to any cell going down (most of our processes handle that case gracefully).
