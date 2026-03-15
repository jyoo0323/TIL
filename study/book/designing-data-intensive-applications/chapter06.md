# Ch.06 Partitioning

---
### Partitioning and Replication
- 파티셔닝은 보통 **복제와 결합됨**
    - 각 레코드는 “정확히 하나의 파티션”에 속함
      - 그러나 fault tolerance를 위해 그 파티션이 여러 노드에 복제될 수 있습니다.
- leader-follower 복제와 결합하면:
    - **파티션마다 leader가 있고**, 그 leader와 follower들이 서로 다른 노드에 분산됨.
    - 한 노드는 “어떤 파티션의 leader”이면서 “다른 파티션의 follower”가 될 수 있음.

---

## Partitioning of Key-Value Data
- 목표:
  - 데이터와 쿼리 부하를 노드들에 균들하게 분배하여, 노드를 10배로 늘리면 처리량도(이론상) 10배 늘리는 것.
- 해결하려는 문제:
  - 분배가 공정하지 않으면 **skew(편향)**됨 -> 한 노드에 너무 많은 데이터가 쌓이게됨.
  - 특정 파티션/노드에 쿼리가 몰리면 **hot spot**(과열 지점)이 됩니다.
- **큰 단점:** 
  - 어느 노드에 데이터가 있는지 알 방법이 없다
    - 키-밸류 모델이라면 키를 기준으로 파티션해 어느 파티션에 있는지 알아낼 수 있음 

### Partitioning by Key Range
- 방식:
  - 키의 range를 파티션에 할당
  - 경계(boundary)를 알면, 어떤 키가 어느 파티션인지 쉽게 결정 → 라우팅이 단순.
- 장점:
  - 파티션 내부에서 키를 정렬 상태(sorted order)로 유지 가능 → **range scan**이 쉬워짐.
  - 키를 “concatenated index”처럼 설계하면, 관련 레코드를 한 번에 가져오기 좋다.
    - 예: 센서 데이터에서 키 = timestamp(YYYY-MM-DD-HH-mm-ss) → 월별/구간 조회가 쉬움.
- 단점(중요): 
  - hot spot 노드가 생길수 있다
    - 키가 시간처럼 단조 증가하면 `오늘의 write 파티션`에 쓰기가 몰림 -> **hot spot**이 발생.
  - 완화 예:
    - timestamp만 쓰지 말고, **sensor name + timestamp** 처럼 prefix를 둬서 먼저 센서로 분산 후 시간 정렬.
      - 트레이드오프: 여러 센서의 시간 범위를 한 번에 읽으려면 센서별로 range query를 여러 번 해야 함.

### Partitioning by Hash of Key
- key-range의 hot spot/편향 위험 때문에, 많은 시스템은 해시 기반 파티셔닝을 사용.
  - 해시 함수는 입력의 편향을 “균일 분포”로 바꾸는 역할.
  - 분산 파티셔닝에서 해시는 암호학적으로 강할 필요는 없고(목표가 보안이 아니라 분산),
    - 예: MongoDB/Cassandra는 MD5, Voldemort는 FNV 계열.
      - 주의: 언어 내장 hash(Object.hashCode 등)는 프로세스마다 달라질 수 있어 분산 파티셔닝에 부적합할 수 있음.
- 구현:
    - key 대신 **hash(key)의 범위**를 파티션에 할당.
    - 파티션 경계는 균등하게 잡거나(균등 구간), 의사난수로 잡기도 함(`hash partitioning`)
      - CDN 같은 캐시 시스템에서 load를 고르게 분산하기 위한 방법
        - consistent는 리밸런싱 시 재배치량을 줄이는 성질을 뜻한다
      - 용어 혼동이 있을 수 있어 consistent hashing은 쓰지 않는다 함
- 단점:
    - key-range의 정렬 속성을 잃어서 **range query가 비효율**.
    - 해시 파티셔닝 모드에서는 범위 조회가 모든 파티션으로 퍼져(scatter)야 할 수 있음.
- 하이브리드:
  - compound primary key (Cassandra):
    - 프라이머리키를 여러 컬럼을 사용해 첫 컬럼을 해시로 파티션을 결정
    - 나머지 컬럼은 파티션 내부 정렬/인덱싱에 사용 → “파티션 내부 범위 조회”를 가능하게 함.

### Skewed Workloads and Relieving Hot Spots
- 해시 파티셔닝도 hot spot을 완전히 막지는 못힘.
  - 모든 요청이 동일한 key로 몰리면, 그 key의 해시도 동일 → 결국 같은 파티션으로 라우팅.
  - ex) 소셜미디아에 유명인 계정/게시물 하나에 쓰기(댓글 등)가 폭주하면 특정 key(게시물 ID 등)가 과열.
- 현재 대부분 시스템은 이런 “극단적 편향”을 자동으로 보정하지 못하므로 **애플리케이션 책임**이 큼.
- 어플리케이션에서의 완화 방법:
  - hot key에 랜덤 suffix/prefix를 붙여 **100개 키로 분산**(예: 2자리 난수).
  - 트레이드오프:
    - 쓰기는 분산되지만, 읽기는 100개 키를 읽고 합쳐야 하므로 비용 증가.
  - **운영 포인트: 모든 키에 적용하면 오버헤드만 커짐 → “어떤 키가 hot key인지”를 추적/관리해야 함.**

---

## Partitioning and Secondary Indexes
- primary key로만 접근하면:
  - key → partition을 결정 → 해당 노드로 라우팅하면 끝.
- secondary index가 있으면:
  - “값(value)”는 다수 레코드를 가리키고 그 레코드들이 여러 파티션에 흩어져 있을 수 있음
  - 인덱스가 파티션과 자연스럽게 매핑되지 않음.
- 파티셔닝 + secondary index 조합은 구현/운영이 복잡해지고, 읽기·쓰기 비용의 트레이드오프가 크게 발생
- document-based & term-based partitioning으로 접근 

### Partitioning Secondary Indexes by Document
- 아이디어:
  - 파티션을 document ID(=primary key)로 나눴다면, 각 파티션은 **자기 파티션 내 문서들만** 대상으로 secondary index를 유지.
  - 즉, 파티션마다 독립적인 인덱스 (local index)
- 장점:
  - 쓰기에 강함:
    - 특정 문서를 추가/수정/삭제할 때 **해당 문서가 있는 파티션만** 업데이트하면 됨.
- 단점:
  - 읽기에 약함:
    - 같은 파티션에 해당 인덱스를 가진 값이 다 있을 보장이 없음.
      - ex: color=red 인 문서들이 여러 파티션에 흩어져 있으므로, red 검색은 **모든 파티션에 질의(scatter) 후 결과를 합치는(gather)** 방식이 됨.
        - scatter/gather는 tail latency를 악화시키기 쉬워 읽기 비용이 비싸짐.
- 읽기시에 단점이 명확하나 그럼에도 널리 쓰임 MongoDB, Riak, Cassandra, Elasticsearch, SolrCloud, VoltDB 등.
- 운영 포인트: 디비 벤더들은 “secondary index query가 단일 파티션에서 처리되도록 파티션 키를 설계"하라고 하지만 필터 조건이 여러 개(예: color + make)로 복잡해지면 현실적으로 어려운 경우가 많음.
  - 현실적으로는 어떻게 접근할까?

### Partitioning Secondary Indexes by Term
- 아이디어:
  - local index 대신, **모든 파티션 데이터를 포괄하는 global index**를 만든다.
    - global index를 한 노드에 몰면 병목이므로, index 자체도 파티셔닝해야 함.
- term-partitioned index:
  - secondary index의 `값(용어/term)`이 어느 파티션에 들어갈지 결정.
  - term 자체로 range 파티셔닝할 수도 있고, term의 hash로 분배할 수도 있음.
  - [12장](chapter12.md) 에서 구현을 다뤄볼 예정
- 장점:
  - 읽기에 강함:
    - 찾고 싶은 term이 들어있는 인덱스 파티션 한 곳만 조회하면 되므로,
    - local index의 scatter/gather보다 효율적일 수 있음.
- 단점:
  - 쓰기에 약함:
    - 문서 하나를 쓰면, 그 문서의 여러 term이 서로 다른 인덱스 파티션에 있을 수 있어
    - **여러 인덱스 파티션을 동시에 업데이트**해야 함 → 느리고 복잡.
    - 이상적으로는 “쓰기 즉시 인덱스 반영”이 좋지만, term-partitioned index에서 이를 보장하려면 관련 파티션들 간 **분산 트랜잭션**이 필요할 수 있음([7장](chapter07.md)/[9장](chapter09.md)).
    - 현실에서는 global secondary index 업데이트가 **비동기(async)** 인 경우가 많아, 인덱스에 “잠깐” 최신 반영이 안 된 상태(=index lag)가 생길 수 있다.

---

## Rebalancing Partitions
- 리밸런싱이 필요한 이유:
  - 처리량 증가 → CPU 추가
  - 데이터 증가 → 디스크/RAM 추가
  - 노드 장애 → 다른 노드가 책임 takeover
- 리밸런 (재균형화): 노드 간에 데이터/요청 부하를 옮겨 **부하를 다시 고르게 만드는 과정**
- 일반적 요구사항:
  - 리밸런싱 후 부하가 공정하게 분배될 것
  - 리밸런싱 중에도 읽기/쓰기 계속 받을 것
  - 필요한 만큼만 데이터 이동(최소 이동)하여 속도/네트워크·디스크 I/O 부담을 줄일 것

### Strategies for Rebalancing
- 파티션을 노드에 할당하는 방식은 여러 가지가 있고,
  - 운영 난이도/성능/유연성이 각각 다름.

#### How not to do it: hash mod N
- hash(key) % N 으로 “노드 번호”를 바로 만들면 쉬워 보입니다.
- 문제:
  - 노드 수 N이 바뀌는 순간, **대부분의 키가 다른 노드로 이동**해야 합니다.
    - 노드 추가/삭제마다 대규모 데이터 이동이 필요 → 리밸런싱 비용 증가.
    - ex: hash=123456:
      - 노드갸 10개일 떄 mod 10 → 6번 노드 
      - 노드가 11개일 때 mod 11 → 3번 노드 
- “필요 이상으로 데이터가 움직이지 않는” 방식이 필요.

#### Fixed number of partitions
- 노드 수보다 훨씬 많은 파티션을 **처음부터 많이 만들어** 두고,
- 각 노드에 여러 파티션을 할당.
  - 노드 추가 시:
    - 새 노드가 기존 노드들로부터 “몇 개 파티션씩” 훔쳐오듯 가져와서 균등 분배.
- 특징:
  - key → partition 매핑은 그대로, **partition → node 할당만 변경**.
    - 파티션 전체를 다른 노드로 이동.
  - 데이터 전송에는 시간이 걸리므로, 전송 중에는 기존 할당을 기반으로 처리.
- 하드웨어 스펙이 다른 경우, 강한 노드에 더 많은 파티션을 줘서 부하를 더 맡길 수도 있음.
- 단점/운영 포인트:
  - 파티션 수는 보통 초기 설정 후 고정 → 미래 노드 확장 고려해 충분히 크게 잡아야 함.
    - 하지만 파티션이 너무 많으면 관리 오버헤드 증가.
  - 파티션 수가 너무 작으면 오버헤드가 커짐.
    - 데이터가 커질수록 파티션도 함께 커짐 → 파티션이 너무 크면 리밸런싱/장애 복구 비용이 커짐.
  - “적당한 파티션 크기”를 맞추기 어렵다는 문제가 있음.

#### Dynamic partitioning
- key-range 파티셔닝에서 “고정 경계 (fixed boundaries)”의 문제:
  - 경계를 잘못 잡으면 한 파티션에 데이터가 몰리고 나머지는 비는 사태가 가능.
- 이를 해결하는 다이나믹 파티셔닝의 방식:
  - 파티션이 일정 크기(ex: HBase default 10GB)를 넘으면 **자동 split**.
    - 반대로 데이터 삭제로 너무 작아지면 **merge**.
  - B-tree 상위 레벨에서 split/merge가 일어나는 것과 유사.
- 장점:
  - 데이터 규모에 따라 파티션 개수가 자동으로 조정 
    - 작은 DB에서는 오버헤드가 작고, 큰 DB에서는 파티션 크기 상한을 유지.
- 단점:
  - 빈 초기의 DB는 “단일 파티션”에서 시작 → 첫 split 전까지는 한 노드에 쓰기가 몰림.
    - pre-splitting(초기 파티션 세트를 미리 생성) 을 통해 완화 가능
      - 단, key distribution을 어느 정도 알아야 함.
- 키 범위만이 아닌 hash 파티셔닝에도 적용 가능(ex: MongoDB 2.4+).

#### Partitioning proportionally to nodes (partitions per node)
- 앞의 두 방식:
  - dynamic: 파티션 수가 “데이터 크기”에 비례
  - fixed: 파티션 크기가 “데이터 크기”에 비례
  - 둘 다 `파티션 수는 노드 수와 무관`
- 노드 수에 비례하는 방식:
  - Cassandra/Ketama 등: 노드당 고정된 수의 파티션 할당
- 새 노드 추가시:
  - 기존 파티션 몇 개를 무작위로 골라 split 후, 절반씩 가져감.
  - 랜덤이라 불공정할 수 있지만, 파티션 수가 충분히 많으면 평균적으로 균등해짐(ex: Cassandra 기본 256 partitions/node).
    - Cassandra 3.0에서는 불공정 split을 피하는 대안 알고리즘 도입.
  - 노드가 늘면 파티션이 더 쪼개져 각 파티션 크기가 다시 작아짐 
    - 파티션 크기를 비교적 안정적으로 유지 가능.
- 이 방식은 hash 기반이어야 랜덤 경계 선택이 가능 
  - 원래 consistent hashing 정의와 가장 가까운 형태로 볼 수 있음.

### Operations: Automatic or Manual Rebalancing
- 자동 vs 수동은 스펙트럼:
  - 완전 자동: 시스템이 스스로 “언제/얼마나/어디로” 옮길지 결정
  - 완전 수동: 관리자가 명시적으로 재할당
  - 중간형: 시스템이 제안(suggest)하고 관리자가 승인(commit) (ex: Couchbase/Riak/Voldemort)
- 완전 자동의 위험:
  - 리밸런싱은 고비용 작업 → 잘못하면 네트워크/노드에 과부하를 주고 전체 성능을 떨어뜨릴 수 있음.
  - 자동 장애 감지와 결합하면 더 위험:
    과부하로 느려진 노드를 “죽었다”고 판단 → 리밸런싱으로 더 많은 일을 떠넘김 → 상태 악화 → 연쇄 장애.
- 그래서 종종 “사람이 루프에 들어가는” 운영이 안정성에 도움이 됩니다(대응은 느리지만 불필요한 surprise 방지).

---

## Request Routing
- 문제:
  - 클라이언트가 key=foo 읽기/쓰기를 하려면 **어느 노드에 붙어야 하는가?**
  - 리밸런싱으로 partition→node 매핑이 바뀌므로, 누군가는 최신 매핑을 알아야 함.
- 이것은 더 일반적으로 **service discovery** 문제. (디비만의 문제가 아님)
- 대표 3가지 접근:
  1) 클라이언트는 아무 노드나 접속 → 그 노드가 담당 파티션이면 처리, 아니면 담당 노드로 forwarding
  2) routing tier(파티션-aware LB)로 먼저 보냄 → routing tier가 담당 노드로 전달
  3) 클라이언트가 파티션/노드 매핑을 알고 직접 해당 노드로 접속
- routing 결정 주체(각 노드/라우팅 티어/클라이언트)가 **파티션 할당 변경 정보를 어떻게 일관되게 알 수 있나?**
    - 이에 사용되는 프로토콜등을 [9장](chapter09.md) 에서 다룰 예정
- coordination service 를 사용하는 방법:
  - ZooKeeper 같은 coordination service에 “클러스터 메타데이터(파티션→노드 매핑)”를 authoritative하게 보관.
  - 노드가 등록하고, 라우팅 티어(2번 방법)/클라이언트(1번/3번 방법)는 구독(subscribe)하여 변경 시 알림을 받음.
    - ex: LinkedIn Espresso(Helix+ZooKeeper), HBase/SolrCloud/Kafka 등에서도 zookeeper를 사용.
      - MongoDB는 config server + mongos 라우팅 티어 구조로 유사한 역할을 수행.
  - Cassandra/Riak: 노드 간 **gossip**로 클러스터 상태 전파
    - 요청은 아무 노드나 받아서 forwarding(접근 1)
    - 외부 코디네이션 서비스 의존을 줄이지만, 노드 쪽 복잡성 증가.
- 기타:
  - 클라이언트가 접속할 IP 목록은 DNS로도 충분한 경우가 많음(파티션 매핑만큼 자주 바뀌지 않기 때문).

### Parallel Query Execution (MPP 맛보기)
- 지금까지는 “단일 key read/write” 중심의 단순 접근(또는 scatter/gather).
- 분석(analytics)용 MPP DB는 훨씬 복잡한 쿼리를 병렬로 실행:
  - join/filter/group/aggregation이 섞인 쿼리를
  - optimizer가 여러 실행 stage로 쪼개고,
  - 여러 노드에서 병렬 실행(특히 대규모 scan에서 큰 이점).
- 자세한 병렬 쿼리 실행 기법은 [10장](chapter10.md)에서 다룰 예정

---

- 파티셔닝의 목적은 데이터/쿼리 부하를 여러 머신에 고르게 분산하여 **hot spot을 피하고** 확장성을 얻는 것.
- 주요 파티셔닝 2가지:
  - key-range: range query에 유리하지만 hot spot 위험. 
    - 보통 dynamic split로 리밸런싱.
  - hash: 부하 분산에 유리하지만 range query가 비효율. 
    - 고정 파티션/동적 파티션 모두 가능.
  - 하이브리드(복합 키)도 가능.
- secondary index와의 결합:
  - local(document-partitioned): 
    - 쓰기 쉬움
    - 읽기 scatter/gather 비쌈
  - global(term-partitioned): 
    - 읽기 좋음
    - 쓰기 복잡/느림
- 라우팅:
  - 파티션을 나눴으면 “어디로 보낼지”가 필수 문제
    - 코디네이션(ZooKeeper 등) 또는 gossip/forwarding 구조가 사용됨.

