# Ch.09 Consistency and Consensus

- [8장](chapter08.md)에서 다룬 것처럼 분산 시스템에서는 많은 것들이 잘못될 수 있다. 결함을 처리하는 가장 단순한 방법은 서비스 전체를 실패시키고 에러 메시지를 보여주는 것이지만, 이것이 용납되지 않는다면 결함을 _감내(tolerating)_ 하는 방법을 찾아야 한다.
  - 내부 컴포넌트 일부에 결함이 있어도 서비스가 올바르게 동작하도록 유지하는 것
- 이번 챕터에서는 결함 내성 분산 시스템을 구축하기 위한 알고리즘과 프로토콜의 예시들을 다룬다.
  - [8장](chapter08.md)의 모든 문제가 발생할 수 있다고 가정: 패킷 유실/지연/중복/순서 뒤바뀜, 시계의 근사치, 노드의 일시 정지/크래시 등

----

## **Consistency Guarantees**

- 복제된 데이터베이스에서 두 노드를 같은 시점에 보면 서로 다른 데이터를 볼 가능성이 높다
  - 쓰기 요청이 서로 다른 시간에 서로 다른 노드에 도착하기 때문
- 대부분의 복제된 데이터베이스는 최소한 **eventual consistency**을 제공한다
  - 불일치는 일시적이며 결국 스스로 해소됨 (네트워크 결함도 복구된다고 가정)
  - 그러나 이것은 매우 약한 보장이다
    - 복제본들이 *언제* 수렴하는지에 대해 아무것도 말해주지 않음
    - 값을 쓰고 바로 읽으면 방금 쓴 값을 볼 수 있다는 보장이 없음 -> 읽기가 다른 복제본으로 라우팅될 수 있기 때문
- Eventual consistency는 애플리케이션 개발자에게 어렵다
  - 일반적인 단일 스레드 프로그램에서 변수에 값을 할당하고 바로 읽으면 할당한 값이 나오거나 실패하지, 이전 값이 나오지는 않는다
- 약한 보장만 제공하는 데이터베이스를 사용할 때는 항상 그 한계를 인식하고, 너무 많이 가정하지 않아야 한다
  - 잘 동작하다가 시스템에 결함이 있거나(네트워크 장애 등) 높은 동시성이 발생할 때만 edge case가 드러남 -> 버그를 테스트로 발견하기 어려움
- 이 챕터에서는 데이터 시스템이 제공할 수 있는 **더 강력한 consistency model**들을 볼 예정
  - 더 강한 보장은 공짜가 아님 == 성능이 나쁘거나 결함 내성이 약할 수 있다
    - 그럼에도 더 강한 보장은 올바르게 사용하기가 더 쉽다는 장점은 있음
- **분산 consistency model과 트랜잭션 isolation level의 비교:**
  - 둘 다 다양한 수준의 보장 계층을 가짐
  - Transaction isolation은 주로 동시에 실행되는 트랜잭션 간의 race condition 회피에 관한 것
  - distributed consistency는 주로 지연과 결함이 있는 상황에서 복제본들의 상태를 조율하는 것에 관한 것
  - 그러나 이 두 영역은 실제로 깊이 연결되어 있음

## Linearizability

- **Linearizability**란 _atomic consistency_, _strong consistency_, _immediate consistency_, _external consistency_ 라고도 불리는 일관성 보장 모델이다
- 핵심 아이디어: 시스템이 마치 **데이터의 복사본이 단 하나만 존재하는 것처럼** 보이게 하고, 모든 연산이 원자적(atomic)으로 수행되는 것처럼 만드는 것
  - 실제로는 여러 replica가 존재하더라도, 애플리케이션은 이를 신경 쓸 필요가 없다
- Linearizable 시스템에서는 한 클라이언트가 write를 성공적으로 완료하는 즉시, 데이터베이스에서 읽는 **모든 클라이언트**가 방금 쓰여진 값을 볼 수 있어야 한다
- 단일 데이터 복사본이라는 환상을 유지한다는 것은, 읽히는 값이 **가장 최근의, 최신의(up-to-date)** 값이며 stale 캐시나 replica에서 온 것이 아님을 보장한다는 것
  - 즉, linearizability는 **recency guarantee** (최신성 보장)이다

### What Makes a System Linearizable?

- 기본 아이디어: 시스템이 데이터의 단일 복사본만 있는 것처럼 보이게 하는 것. 
  - 이것이 정확히 무엇을 의미하는지는 세심한 정의가 필요하다
- 분산 시스템 문헌에서 변수 _x_를 **register**라고 부른다
  - 실제로는 key-value store의 하나의 키, 관계형 DB의 한 row, 문서 DB의 한 문서 등이 될 수 있다
- Register는 세 가지 연산을 지원:
  - **read(x) => v**: register x의 값을 읽어 v를 반환
  - **write(x, v) => r**: register x에 값 v를 쓰고 응답 r (ok 또는 error)을 반환
  - **cas(x, v_old, v_new) => r**: atomic **compare-and-set** 연산. register x의 현재 값이 v_old와 같으면 v_new로 원자적으로 설정하고, 다르면 변경 없이 에러 반환
- 동시성과 linearizability 조건:
  - write 시작 **이전**에 완료된 read는 반드시 **이전 값** 을 반환
  - write 완료 **이후**에 시작된 read는 반드시 **새 값** 을 반환 (linearizable 시스템이라면)
  - write와 **concurrent**(동시 발생)한 read는 이전 값 또는 새 값을 반환할 수 있음
- **추가 제약**: 어떤 read가 새 값을 반환했다면, 그 이후의 **모든** read(같은 클라이언트든 다른 클라이언트든)도 반드시 새 값을 반환해야 한다
  - linearizable 시스템에서는 write 연산의 시작과 끝 사이 어딘가에서 값이 원자적으로 전환되는 **단일 시점**이 존재한다고 상상할 수 있다
- **Linearizability의 핵심 요구사항**: 연산 마커들을 연결하는 선이 시간 순서대로 항상 **앞으로만(왼쪽에서 오른쪽으로)** 이동해야 하며, 절대 뒤로 가서는 안 된다

#### Linearizability vs Serializability

- 두 단어가 비슷해 보이지만 **완전히 다른 보장**이다:
  - **Serializability**: _Transaction_ 의 격리 속성. 여러 객체에 대한 읽기/쓰기가 어떤 직렬 순서로 실행된 것처럼 동작. 실제 실행 순서와 다를 수 있음
  - **Linearizability**: 개별 _register_ (개별 객체)에 대한 recency guarantee. 트랜잭션으로 묶지 않음. 실시간 순서(recency) 보장
- 두 가지를 모두 제공하는 것을 **strict serializability** 또는 **strong one-copy serializability (strong-1SR)** 라고 한다
  - **Two-Phase Locking (2PL)** 이나 **actual serial execution** 기반의 serializability 구현은 보통 linearizable
  - 그러나 **Serializable Snapshot Isolation (SSI)** 는 linearizable하지 않음: snapshot은 최신 write를 포함하지 않을 수 있으므로

### Relying on Linearizability

- linearizability가 시스템의 올바른 동작을 위해 **필수적인** 영역들은 아래 3개의 케이스가 있다

- **_Locking and leader election_**
  - Single-leader replication에서 leader가 단 하나만 존재해야 한다 (split brain 방지)
    - lock을 사용해 leader를 선출할 수 있으며, 이 lock의 구현은 반드시 **linearizable**해야 한다
  - **Apache ZooKeeper**, **etcd** 같은 coordination service가 분산 lock과 leader election을 구현하는 데 자주 사용됨
    - consensus 알고리즘을 사용하여 fault-tolerant한 방식으로 linearizable 연산을 구현

- _**Constraints and uniqueness guarantees**_
  - 두 사람이 동시에 같은 사용자명을 등록하려 할 때 하나만 성공하고 다른 하나는 에러를 반환하려면 **linearizability가 필요**하다
    - 이는 사실상 lock과 유사: atomic **compare-and-set**
  - 은행 잔고가 음수가 되지 않도록 보장, 재고보다 많이 판매하지 않도록 보장 등도 모두 **단일 최신 값(single up-to-date value)** 이 필요
  - 관계형 데이터베이스의 **hard uniqueness constraint**는 linearizability를 요구함

- **_Cross-channel timing dependencies_**
  - 시스템에 **추가적인 통신 채널**이 있을 때 linearizability 위반이 감지됨
    - ex: 파일 스토리지에 이미지를 저장하고 message queue에 리사이즈 지시를 넣는 경우
      - message queue가 파일 스토리지 내부 replication보다 빠를 수 있어 resizer가 이미지를 가져올 때 이전 버전을 보거나 아예 없을 수 있음
    - 이 문제는 **두 개의 다른 통신 채널** 간의 race condition 때문에 발생

### Implementing Linearizable Systems

- 가장 단순한 답은 실제로 **단일 복사본만 사용**하는 것이지만, fault tolerance가 없어지게 될 것이다.
- [5장](chapter05.md)의 replication 방식별 linearizability 가능 여부:
  - **Single-leader replication** (_potentially_ linearizable): Leader 또는 동기적으로 업데이트된 follower에서 읽으면 잠재적으로 linearizable. 단 snapshot isolation 사용이나 concurrency 버그, 비동기 replication failover 시 문제 가능
  - **Consensus algorithms** (_linearizable_): ZooKeeper, etcd 등이 split brain과 stale replica를 방지하는 조치 포함
  - **Multi-leader replication** (_not_ linearizable): 여러 노드에서 동시에 write를 처리하고 비동기적으로 replicate하므로 일반적으로 linearizable하지 않음
  - **Leaderless replication** (_probably not_ linearizable): strict quorum (w + r > n) 에서도 가변적인 네트워크 지연 때문에 race condition이 발생 가능. Last write wins는 clock skew 문제로 거의 확실히 non-linearizable

#### Linearizability and quorums
- strict quorum이 linearizable해야 할 것 같지만, 가변적인 네트워크 지연 때문에 **race condition이 발생 가능**
  - ex: n=3, w=3, r=2 환경에서 Reader A가 새 값 1을 읽은 후 Reader B가 다른 quorum에서 이전 값 0을 읽을 수 있음
- Dynamo 스타일 quorum을 linearizable하게 만드는 것은 **가능**하지만 성능 비용이 있다 (동기적 read repair 등)
  - **linearizable compare-and-set 연산은 불가능** - consensus 알고리즘이 필요

### The Cost of Linearizability

- Multi-datacenter 배포 시나리오:
  - **Multi-leader replication**: 데이터센터 간 네트워크가 중단되어도 각 데이터센터가 독립적으로 정상 운영 가능
  - **Single-leader replication**: 데이터센터 간 네트워크 중단 시, follower 데이터센터의 클라이언트는 linearizable read/write 불가

#### The CAP theorem
- 트레이드오프:
  - 애플리케이션이 linearizability를 **요구**하고 네트워크 문제로 일부 replica가 단절되면 -> 해당 replica는 요청을 처리할 수 없음 -> **unavailable**
  - 애플리케이션이 linearizability를 **요구하지 않으면** -> 각 replica가 독립적으로 요청 처리 가능 -> 네트워크 장애에도 **available** 유지, 단 linearizable하지 않음
- CAP는 종종 **Consistency, Availability, Partition tolerance: 3개 중 2개를 선택**이라고 표현되지만, 이는 **오해의 소지가 크다**:
  - network partition은 일종의 장애(fault)이므로 선택의 대상이 아님 - 좋든 싫든 **일어난다**
  - 더 정확한 표현: **"Either Consistent or Available when Partitioned"**
- CAP theorem의 형식적 정의는 매우 좁은 범위: **하나의 consistency model(linearizability)**과 **하나의 fault 종류(network partition)**만 고려
  - CAP는 역사적으로 영향력이 있었으나 실제 시스템 설계에 있어 실용적 가치가 적고 **피하는 것이 좋다**

#### Linearizability and network delays
- 현대 멀티코어 CPU의 RAM조차 **linearizable하지 않다**
  - 각 CPU 코어는 자체 메모리 캐시와 store buffer를 가지고 있고, 메모리 접근은 먼저 캐시로 가며 변경사항이 main memory에 비동기적으로 기록됨
  - **memory barrier** 또는 **fence**를 사용하지 않으면 최신 값을 보장받지 못함
- **많은 분산 데이터베이스도 같은 이유로 linearizable 보장을 제공하지 않는다**: fault tolerance보다 **성능 향상**이 주된 목적
- Attiya와 Welch의 증명: linearizability를 원하면 read/write 요청의 응답 시간이 **네트워크 지연의 불확실성에 최소한 비례**해야 한다
  - 더 빠른 linearizability 알고리즘은 존재하지 않지만, 더 약한 consistency 모델은 훨씬 빠를 수 있다

## Ordering Guarantees

- Ordering은 이 책 전반에 걸쳐 반복적으로 등장하는 핵심 개념이다:
  - **Single-leader replication**: 리더의 주요 역할은 replication log에서 **쓰기 순서(order of writes)** 를 결정하는 것 ([5장](chapter05.md))
  - **Serializability**: 트랜잭션들이 어떤 _순차적 순서(sequential order)_ 로 실행된 것처럼 동작하게 보장 ([7장](chapter07.md))
  - **Timestamp와 clock**: 분산 시스템에서 어떤 쓰기가 먼저 발생했는지 판단하려는 시도 ([8장](chapter08.md))
- Ordering, linearizability, consensus 사이에는 **깊은 연결 관계**가 존재한다

### Ordering and Causality

- Ordering이 중요한 이유 중 하나는 **causality(인과성)** 를 보존하기 때문이다
- 이 책 전반에 걸쳐 인과성이 중요했던 사례들:
  - **Consistent Prefix Reads** ([5장](chapter05.md)): 대화에서 질문보다 답변이 먼저 보이는 현상 - 질문과 답변 사이에 _causal dependency_ 가 있음
  - **Snapshot isolation** ([7장](chapter07.md)): consistent snapshot은 **causality와 일관적(consistent with causality)** 이어야 함
  - **Write skew** ([7장](chapter07.md)): 트랜잭션 간 causal dependency 존재 - 현재 누가 on-call인지에 대한 관찰에 기반한 행동
- Causality는 이벤트에 순서를 부과한다: **원인이 결과보다 먼저** 온다

#### The causal order is not a total order

- **Total order**: 임의의 두 요소를 항상 비교 가능
  - ex: 자연수는 totally ordered - 5와 13 중 어느 것이 더 큰지 항상 말할 수 있음
- **Partial order**: 일부 요소는 비교 가능하지만 일부는 _incomparable_
  - ex: 수학적 집합 {a, b}와 {b, c}는 비교 불가 (어느 쪽도 부분집합이 아님)
- 다양한 데이터베이스 일관성 모델에서의 차이:
  - **Linearizability**: 연산의 **total order** 가 존재 - 데이터 복사본이 하나뿐이고 모든 연산이 원자적이므로 임의의 두 연산에 대해 어느 것이 먼저인지 항상 말할 수 있음
  - **Causality**: **partial order** 를 정의 - 인과적으로 관련된 두 이벤트는 순서가 있지만, concurrent한 이벤트는 incomparable
- Concurrency는 타임라인이 분기(branch)하고 병합(merge)되는 것을 의미 - Git의 버전 히스토리와 매우 유사

#### Linearizability is stronger than causal consistency

- Linearizability는 causality를 **_implies_** 한다: linearizable한 시스템은 자동으로 인과성을 올바르게 보존
- 그러나 linearizability에는 **성능과 가용성의 비용**이 따른다
- **Causal consistency는 네트워크 장애에도 가용성을 유지하면서 네트워크 지연으로 느려지지 않는, 가장 강한 일관성 모델이다**
  - CAP theorem이 적용되지 않음
- 실제로 linearizability가 필요해 보이는 많은 시스템이 사실은 **causal consistency만으로 충분**한 경우가 많다

#### Capturing causal dependencies

- 인과성을 유지하려면 어떤 연산이 다른 연산보다 _happened before_ 인지 알아야 한다
- Causal dependency를 파악하기 위해 노드의 **"지식(knowledge)"** 을 추적해야 함
- 인과적 순서를 파악하는 기법은 [5장](chapter05.md)의 **concurrent writes 감지** 와 유사
  - 다만 causal consistency는 단일 키가 아닌 **전체 데이터베이스**에 걸쳐 causal dependency를 추적해야 함
  - **Version vector**를 일반화하여 사용 가능

### Sequence Number Ordering

- 모든 causal dependency를 명시적으로 추적하는 것은 비현실적일 수 있다 (오버헤드가 큼)
- 더 나은 방법: **sequence number** 또는 **timestamp**를 사용하여 이벤트를 정렬
  - 여기서 timestamp는 물리적 시계(time-of-day clock)가 아닌 **logical clock**에서 올 수 있음 ([8장](chapter08.md))
- Sequence number/timestamp는 **compact**(수 바이트)하고, **total order**를 제공
- **Causality와 consistent한 total order** 생성 가능: 연산 A가 인과적으로 B보다 먼저 발생했다면, A의 sequence number가 B보다 작음
- **Single-leader replication**에서는 리더가 단순히 카운터를 증가시키며 각 연산에 monotonically increasing sequence number를 부여

#### Noncausal sequence number generators

- Single leader가 없는 경우 sequence number 생성이 더 어려움
- 실무에서 사용되는 방법들:
  - **노드별 독립 생성**: 두 노드가 각각 홀수/짝수 번호 사용
  - **Physical clock timestamp 부착**: last write wins 방식에서 사용 ([8장](chapter08.md))
  - **Sequence number 블록 사전 할당**: 노드 A는 1~1,000, 노드 B는 1,001~2,000 범위
- 이 세 방법 모두 **causality와 consistent하지 않다**는 문제가 있다

#### Lamport timestamps

- **Lamport timestamp**: 1978년 Leslie Lamport가 제안, causality와 consistent한 sequence number를 생성하는 간단한 방법
- 구조: **(counter, node ID)** 쌍
  - 각 노드는 고유 식별자를 가지고, 처리한 연산 수의 카운터를 유지
  - 두 timestamp의 counter 값이 다르면 큰 쪽이 더 큰 timestamp, counter가 같으면 node ID가 큰 쪽이 더 큰 timestamp
- **핵심 아이디어**: 모든 노드와 클라이언트가 지금까지 본 **maximum counter 값**을 추적하고, 모든 요청에 이 maximum을 포함
  - 노드가 자신의 counter보다 더 큰 counter 값을 가진 요청/응답을 받으면, 즉시 자신의 counter를 그 maximum으로 증가
- **Lamport timestamp vs Version vector**:
  - Version vector: 두 연산이 concurrent인지 인과적으로 dependent인지 **구분 가능**
  - Lamport timestamp: 항상 **total ordering을 강제** - concurrent인지 dependent인지 구분 불가. 대신 더 **compact**

#### Timestamp ordering is not sufficient

- Lamport timestamp가 causality와 consistent한 total order를 정의하지만, 분산 시스템의 많은 문제를 해결하기에는 **충분하지 않다**
- ex: 사용자명 고유성(uniqueness) 제약
  - Lamport timestamp으로 같은 username의 두 계정 중 낮은 timestamp을 가진 쪽을 승자로 선택하면 될 것 같지만 이 방법은 모든 연산을 수집한 **후에(after the fact)** 만 승자를 결정할 수 있음
  - 다른 노드가 아직 생성 중인 연산이 total order의 다양한 위치에 끼어들 수 있음
- **결론**: username 같은 uniqueness 제약을 구현하려면 단순히 total ordering만으로는 부족하고, **그 순서가 확정되는 시점을 알아야** 한다 -> 이 아이디어가 바로 **total order broadcast**

### Total Order Broadcast

- 분산 시스템에서 모든 노드가 **동일한 total ordering에 합의**하는 것은 어려움
- Single-leader replication은 리더가 단일 CPU 코어에서 연산을 순서화하여 total order를 결정
  - 과제: 리더가 처리 가능한 것 이상의 throughput이 필요하거나, 리더 장애 시 failover 처리
- Total order broadcast (또는 _atomic broadcast_): 노드 간 **메시지 교환 프로토콜**로, 두 가지 safety property를 항상 만족해야 한다:
  - **Reliable delivery**: 메시지가 한 노드에 전달되면, _모든_ 노드에 전달됨 (유실 없음)
  - **Totally ordered delivery**: 메시지가 모든 노드에 **같은 순서**로 전달됨

#### Using total order broadcast

- ZooKeeper, etcd 같은 **consensus 서비스**가 실제로 total order broadcast를 구현
- **Database replication**: 모든 메시지가 DB 쓰기를 나타내고, 모든 replica가 같은 순서로 쓰기를 처리하면 replica들이 일관적으로 유지됨 -> **state machine replication**
- Total order broadcast의 **중요한 특성**: 메시지 전달 시점에 순서가 **확정(fixed)** 됨
  - 이후 메시지가 이미 전달된 상태에서 이전 위치에 메시지를 소급 삽입할 수 없음
  - 이것이 total order broadcast가 **timestamp ordering보다 강력한** 이유
- Total order broadcast는 **로그(log)** 를 만드는 것과 같다
- **Lock service / fencing token** 구현에도 유용: 잠금 요청이 로그에 append되고, 순서대로 번호가 매겨짐 -> monotonically increasing sequence number가 fencing token 역할 (ZooKeeper의 `zxid`)

#### Implementing linearizable storage using total order broadcast

- Total order broadcast와 linearizability는 **밀접하지만 동일하지는 않다**
  - Total order broadcast: 메시지가 고정된 순서로 비동기 전달, **_언제_** 전달될지 보장 없음
  - Linearizability: **recency guarantee** - 읽기가 가장 최근에 쓰여진 값을 반환하도록 보장
- **Linearizable compare-and-set을 total order broadcast로 구현하는 방법** (append-only log 사용):
  1. 원하는 username을 주장(claim)하는 메시지를 로그에 **append**
  2. 로그를 읽으며, 자신이 append한 메시지가 자신에게 **다시 전달될 때까지 대기**
  3. 원하는 username에 대한 메시지 중 **첫 번째가 자신의 메시지인지 확인** - 자신이 첫 번째이면 성공(commit), 아니면 abort
- 이 절차는 **linearizable writes**는 보장하지만, **linearizable reads**는 보장하지 않음
  - **Linearizable reads를 위한 옵션들**:
    - 로그에 메시지를 append하고 전달될 때까지 기다린 후 읽기 수행 (etcd의 quorum read)
    - 로그에서 최신 메시지의 위치를 linearizable하게 조회하고 해당 위치까지 기다림 (ZooKeeper의 `sync()`)
    - 쓰기에 대해 동기적으로 업데이트되는 replica에서 읽기 (chain replication)

#### Implementing total order broadcast using linearizable storage

- 반대 방향도 가능: **linearizable storage로 total order broadcast를 구현**할 수 있다
- 방법: atomic **increment-and-get** 연산을 지원하는 linearizable register를 사용
  - 모든 메시지에 대해 linearizable integer를 increment-and-get하고, 얻은 값을 sequence number로 부착
- **Lamport timestamp와의 차이**: 
  - linearizable register에서 얻는 숫자는 **gap이 없는 연속적 시퀀스**를 형성
    - 노드가 메시지 4를 전달하고 다음에 sequence number 6인 메시지를 받으면, 메시지 5를 받을 때까지 **대기해야** 함
- Linearizable cas register와 total order broadcast는 모두 **consensus와 equivalent** 함이 증명 가능
  - 즉, 이 중 하나를 풀 수 있으면 나머지도 풀 수 있다

## Distributed Transactions and Consensus

- **Consensus**는 분산 컴퓨팅에서 가장 중요하고 근본적인 문제 중 하나이다: _여러 노드가 무언가에 동의하게 만드는 것_
- Consensus가 중요한 상황들:
  - **Leader election**: single-leader replication에서 모든 노드가 누가 리더인지 합의해야 한다. 합의가 없으면 split brain 발생
  - **Atomic commit**: 여러 노드/파티션에 걸친 트랜잭션에서 모든 노드가 트랜잭션의 결과에 동의해야 한다 ([7장](chapter07.md))

#### The Impossibility of Consensus (FLP result)

- **FLP result** (Fischer, Lynch, Paterson): 노드가 crash할 위험이 있는 분산 시스템에서는 항상 합의에 도달할 수 있는 알고리즘이 존재하지 않음을 증명
  - 단, **asynchronous system model** (클록이나 타임아웃을 사용할 수 없는 모델)에서 증명된 것
  - 타임아웃이나 crash 의심 메커니즘을 허용하면 합의는 해결 가능해짐
  - 따라서 이론적으로는 불가능하지만, 실제 분산 시스템에서는 합의를 달성할 수 있다

### Atomic Commit and Two-Phase Commit (2PC)

- [7장](chapter07.md)에서 배운 것처럼, 트랜잭션 atomicity의 목적은 여러 쓰기 도중 문제가 발생했을 때 단순한 시맨틱을 제공하는 것
  - 트랜잭션의 결과는 **commit** (모든 쓰기가 durable) 또는 **abort** (모든 쓰기가 롤백) 둘 중 하나

> **2PC와 2PL을 혼동하지 말 것!** Two-phase _commit_ (2PC)과 two-phase _locking_ (2PL, [7장](chapter07.md))은 완전히 다른 개념이다. 

> 2PC는 분산 데이터베이스에서 atomic commit을 제공하고, 2PL은 serializable isolation을 제공한다.

#### From single-node to distributed atomic commit

- **단일 노드**에서의 atomicity: **commit record를 쓰는 순간**이 트랜잭션의 커밋/abort를 결정하는 _point of no return_
- **여러 노드**가 관여하는 경우: 단순히 모든 노드에 커밋 요청을 보내는 것으로는 충분하지 않다
  - 일부 노드에서 constraint violation, 네트워크 유실, 커밋 전 crash 등이 발생할 수 있음
  - 일부 노드는 커밋하고 다른 노드는 abort하면 노드 간 불일치 발생

#### Introduction to two-phase commit

- **Two-phase commit (2PC)**: 여러 노드에 걸쳐 atomic 트랜잭션 커밋을 달성하기 위한 알고리즘
- **coordinator** (또는 **transaction manager**)라는 새로운 컴포넌트를 도입
- **2PC 기본 흐름**:
  - 애플리케이션이 여러 participant 노드에서 데이터를 읽고 쓴다
  - **Phase 1 (Prepare)**: coordinator가 모든 participant에게 prepare 요청을 보내고, 커밋할 수 있는지 물어봄
  - **Phase 2 (Commit/Abort)**: 모든 participant가 "yes"로 응답하면 commit 요청, 하나라도 "no"면 abort 요청

#### A system of promises

- 2PC 세부 프로세스:
  1. Coordinator로부터 **globally unique한 transaction ID**를 요청
  2. 각 participant에서 single-node 트랜잭션을 시작하고, global transaction ID를 부착
  3. Coordinator가 모든 participant에게 **prepare** 요청. 하나라도 실패/타임아웃이면 전체 abort
  4. Participant가 prepare 요청을 받으면, **어떤 상황에서든 반드시 커밋할 수 있음을 확인** 후 "yes" 응답 = **약속(promise)**. _participant는 abort할 권리를 포기하지만, 아직 실제로 커밋하지는 않음_
  5. Coordinator가 모든 응답을 받고 **최종 결정**을 **transaction log에 디스크에 기록** = **commit point**
  6. 모든 participant에게 commit/abort 요청. 실패/타임아웃 시 **성공할 때까지 영원히 재시도**

#### Coordinator failure

- Participant가 "yes"를 투표한 **후에** coordinator가 crash하면 -> participant는 커밋할지 abort할지 알 수 없다 = **_in doubt_ 또는 _uncertain_ 상태**
  - 타임아웃도 도움이 되지 않는다 (일방적 결정은 다른 participant와의 불일치 야기)
  - **유일한 해결책은 coordinator가 복구될 때까지 기다리는 것**
- 2PC는 coordinator 복구를 기다리며 멈출 수 있으므로 **blocking** atomic commit 프로토콜이라 불림

#### Three-phase commit

- **Three-phase commit (3PC)**: 2PC의 대안으로 제안된 알고리즘
  - 그러나 **bounded delay를 가진 네트워크**와 **bounded response time을 가진 노드**를 가정
  - 대부분의 실제 시스템에서는 이 가정이 성립하지 않으므로 atomicity를 보장할 수 없다
  - 이런 이유로 coordinator failure 문제에도 불구하고 2PC가 계속 사용된다

### Distributed Transactions in Practice

- Distributed transaction은 **엇갈린 평판**을 가진다
  - 한편으로는 중요한 안전성 보장을 제공, 다른 한편으로는 운영 문제를 야기하고 성능을 저해
  - ex: MySQL에서 distributed transaction은 single-node transaction보다 **10배 이상 느리다**고 보고됨

#### Distributed transaction의 두 가지 유형

- **Database-internal distributed transactions**: 동일한 데이터베이스 소프트웨어를 사용하는 노드 간의 트랜잭션 - 꽤 잘 동작
- **Heterogeneous distributed transactions**: 서로 다른 기술의 participant 간 트랜잭션 - 훨씬 더 도전적

#### Exactly-once message processing

- ex: message queue의 메시지를 처리하는 데이터베이스 트랜잭션이 성공한 경우에만 메시지를 acknowledged로 처리
  - 메시지 전달이나 데이터베이스 트랜잭션 중 하나라도 실패하면 둘 다 abort
  - 이를 통해 메시지가 **effectively exactly once** 처리됨을 보장

#### XA transactions

- **X/Open XA** (_eXtended Architecture_): heterogeneous 기술 간 two-phase commit을 구현하기 위한 **표준** (1991년 도입)
  - 지원 DB: PostgreSQL, MySQL, DB2, SQL Server, Oracle
  - 지원 message broker: ActiveMQ, HornetQ, MSMQ, IBM MQ
- XA는 네트워크 프로토콜이 **아니다** -- transaction coordinator와 인터페이스하기 위한 **C API**
  - Java EE에서는 **JTA** (Java Transaction API)를 통해 구현
- **Transaction coordinator가 XA API를 구현**: 실제로는 애플리케이션과 같은 프로세스에 로드되는 라이브러리인 경우가 많다
  - 애플리케이션 프로세스가 crash하면 coordinator도 함께 죽어 in doubt 상태 발생

#### Holding locks while in doubt

- In-doubt 트랜잭션이 lock을 잡고 있으면 심각한 문제 발생:
  - Coordinator가 crash하고 재시작하는 데 20분이 걸리면 -> 20분 동안 lock이 유지
  - Coordinator의 로그가 완전히 유실되면 -> **lock이 영원히 유지**될 수 있다
  - 결과적으로 in-doubt 트랜잭션이 해결될 때까지 **애플리케이션의 큰 부분이 사용 불가**해질 수 있다

#### Recovering from coordinator failure

- 실제로는 **orphaned** in-doubt 트랜잭션이 발생 - coordinator가 결과를 결정할 수 없는 트랜잭션
  - DB 서버를 재부팅해도 해결되지 않음: 올바른 2PC 구현은 재시작 후에도 in-doubt 트랜잭션의 lock을 유지
- **유일한 해결책은 관리자가 수동으로** 각 participant를 조사하고 동일한 결과를 적용하는 것
- **Heuristic decisions**: coordinator 없이 participant가 일방적으로 in-doubt 트랜잭션을 커밋/abort하는 비상 탈출구
  - "아마도 atomicity를 깨뜨리는" 완곡한 표현 
  - 오직 재앙적 상황을 벗어나기 위한 것

#### Limitations of distributed transactions

- XA transaction의 심각한 **운영 문제**:
  - **Coordinator가 single point of failure**: coordinator 장애 시 다른 application server들이 in-doubt 트랜잭션의 lock 때문에 블로킹
  - **Stateless 애플리케이션 모델과 충돌**: coordinator의 로그가 application server의 durable한 상태의 중요 부분이 되므로 더 이상 stateless하지 않게 됨

### Fault-Tolerant Consensus

- Consensus란 **여러 노드가 무언가에 합의하는 것**
  - ex: 여러 사람이 동시에 비행기의 마지막 좌석을 예약하거나, 같은 사용자명으로 계정을 등록하려 할 때
- Consensus 알고리즘은 다음 네 가지 속성을 만족해야 한다:
  - **Uniform agreement**: 어떤 두 노드도 서로 다른 결정을 내리지 않는다
  - **Integrity**: 어떤 노드도 두 번 결정하지 않는다
  - **Validity**: 노드가 값 *v*를 결정했다면, *v*는 어떤 노드가 제안한 값이다
  - **Termination**: 크래시하지 않은 모든 노드는 결국 어떤 값을 결정한다
- Termination은 **내결함성(fault tolerance)** 을 형식화한 속성 - _liveness_ 속성이고, 나머지 세 가지는 _safety_ 속성 ([8장 참고](chapter08.md))
  - Termination을 보장하려면 **과반수(majority) 이상의 노드가 정상 동작**해야 한다

#### Consensus 알고리즘과 Total Order Broadcast

- 대표적인 fault-tolerant consensus 알고리즘: **Viewstamped Replication (VSR)**, **Paxos**, **Raft**, **Zab**
- 대부분의 알고리즘은 **값의 시퀀스(sequence)** 에 대해 결정 -> **total order broadcast** 알고리즘
- Total order broadcast는 **반복적인 consensus 라운드**와 동치:
  - 각 라운드에서 노드들이 다음에 전달할 메시지를 제안하고, 전체 순서에서 다음으로 전달될 메시지를 결정
- Viewstamped Replication, Raft, Zab은 total order broadcast를 직접 구현하며, Paxos의 경우 이를 **Multi-Paxos**라 한다

#### Single-leader Replication과 Consensus

- [5장](chapter05.md)에서 다룬 single-leader replication은 본질적으로 total order broadcast와 같다
- 핵심은 **리더가 어떻게 선출되느냐**에 있다:
  - 리더를 선출하려면 consensus가 필요하고, consensus를 풀려면 리더가 필요 - **닭과 달걀 문제**
  - 이 순환을 깨는 것이 epoch numbering과 quorum 메커니즘

#### Epoch Numbering과 Quorums

- 모든 consensus 프로토콜은 내부적으로 **epoch number**라는 약한 보장을 제공 (Paxos: _ballot number_, Viewstamped Replication: _view number_, Raft: _term number_)
  - 각 epoch 내에서 리더가 유일하다는 것을 보장
- 현재 리더가 죽은 것으로 판단되면 **투표가 시작**되어 새 리더를 선출하고, epoch number가 단조 증가
- **두 번의 투표 라운드**가 존재:
  1. 리더를 선출하기 위한 투표
  2. 리더의 제안에 대한 투표
- 핵심: 이 두 투표의 quorum이 **겹쳐야(overlap)** 한다
- **2PC와의 차이**:
  - 2PC: coordinator가 선출되지 않으며, **모든 참여자**의 "yes" 투표가 필요
  - Consensus 알고리즘: **과반수의 노드**로부터 투표만 받으면 됨 + 새 리더 선출 후 일관된 상태 복구를 위한 **recovery 프로세스** 정의

#### Consensus의 한계

- Consensus 알고리즘은 분산 시스템의 중요한 돌파구이지만 **비용이 따른다**:
  - 제안에 대한 투표 과정은 **동기식 복제(synchronous replication)** 의 일종
  - **엄격한 과반수(strict majority)** 가 필요: 장애 1개 허용 -> 최소 3노드, 2개 허용 -> 최소 5노드
  - 대부분의 consensus 알고리즘은 **고정된 노드 집합**을 가정
  - **네트워크 문제에 민감**: timeout에 의존하여 장애 노드를 감지하므로, 가변적인 네트워크 지연 환경에서 빈번한 리더 선출이 성능을 크게 저하시킬 수 있다
  - 불안정한 네트워크에서도 견고한 consensus 알고리즘 설계는 여전히 **열린 연구 문제**

### Membership and Coordination Services

- ZooKeeper나 etcd 같은 프로젝트는 "분산 key-value 저장소" 또는 "coordination and configuration 서비스"로 불린다
  - **메모리에 들어갈 수 있는 소량의 데이터**를 보관하도록 설계 (fault-tolerant total order broadcast 알고리즘으로 모든 노드에 복제)
- ZooKeeper는 Google의 **Chubby** lock 서비스를 모델로 하며, 분산 시스템 구축에 유용한 여러 기능을 제공:

#### ZooKeeper가 제공하는 기능

- **Linearizable atomic operations**: atomic compare-and-set 연산으로 lock 구현 가능. 분산 lock은 보통 만료 시간이 있는 **lease**로 구현
- **Total ordering of operations**: 모든 연산에 단조 증가하는 transaction ID(**zxid**)와 version number(**cversion**)를 부여하여 fencing token 제공
- **Failure detection**: long-lived session과 heartbeat 교환. 세션이 만료되면 해당 세션이 보유한 lock 자동 해제 (**ephemeral node**)
- **Change notifications**: 다른 클라이언트가 클러스터에 참여하거나 실패할 때 알림 - 빈번한 polling 없이 변경 사항 감지

#### Allocating work to nodes

- ZooKeeper/Chubby 모델이 잘 작동하는 사례:
  - 여러 인스턴스 중 **하나를 리더/프라이머리로 선출**해야 하는 경우
  - 파티션된 리소스에서 **어떤 파티션을 어떤 노드에 할당**할지 결정하는 경우 ([6장 Rebalancing Partitions 참고](chapter06.md))
- ZooKeeper는 **고정된 소수의 노드(보통 3개 또는 5개)** 에서 실행되어 잠재적으로 많은 수의 클라이언트를 지원
  - coordination 작업의 일부를 **외부 서비스에 "아웃소싱"** 하는 것
- ZooKeeper가 관리하는 데이터는 보통 느리게 변하는 정보 (분~시간 단위로 변경)
  - 고빈도 데이터에는 Apache BookKeeper 같은 도구 사용

#### Service Discovery

- ZooKeeper, etcd, Consul은 **service discovery**에도 자주 사용
  - VM이 지속적으로 생성/소멸되는 클라우드 환경에서 서비스의 네트워크 엔드포인트를 서비스 레지스트리에 등록
- Service discovery 자체는 consensus가 필요 없지만, **leader election**에는 consensus가 필요
  - 일부 consensus 시스템은 **read-only caching replica**를 지원

#### Membership Services

- ZooKeeper와 유사한 시스템은 **membership service**에 대한 오랜 연구의 일부
  - **클러스터에서 현재 활성 상태인(살아 있는) 노드가 어떤 것인지 결정**
  - **consensus와 장애 감지를 결합**하면, 노드들이 어떤 노드를 살아있는 것으로 간주할지 **합의**할 수 있다
  - 비록 죽었다고 선언된 노드가 실제로는 살아있을 수 있지만, 현재 멤버십에 대한 **합의**를 갖는 것 자체가 매우 유용

