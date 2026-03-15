# Part 2 Distributed Data
- main question: 여러 머신이 저장과 검색에 관여했다면 어떻게 될까?
- 분산 데이터 베이스가 필요한 이유:
  - Scalability
    - 하나 DB로 못버틸 로드 (데이터의 양, 읽기/쓰기 로드)
  - Fault tolerance / High Availability
    - 하나의 머신, 네트워크, 혹은 데이터센터 전체가 맛이 갈 경우에도 동작해야함
  - Latency
    - 글로벌 서비스의 경우 가까운 리전의 데이터 센터 선택
### Scaling to Higher Load
- shared-memory architecture (Scale up):
  - 하나의 머신에 더 많은 컴포넌트를 추가해 부하를 견디게하는 방식
- shared-disk architecture
  - 다른 CPU와 메모리를 가지지만, 하나의 디스크를 공유해서 사용하는 방법
    - NAS(Network Attached Storage), SAN(Storage Area Network) 등을 통해 가능
- 위 두 방식은 제한적임.. 락이나 경합이 발생시 확장성이 저하됨
- shared-nothing architecture (scale out)
  - Node: machine or virtual machine running the database software
  - 특별한 하드웨어도 필요없음
  - 이 책에서 중점적으로 다룰 방식
  - 단점:
    - 높은 어플리케이션 복잡도
    - 사용할 수 있는 데이터 모델에 제한을 걸기도 한다
### Replication vs Partitioning
- Replication:
  - 같은 데이터를 여러 노드에 분산시키는 것
    - 다른 지역일 수도 있음
  - availability & performance 에 영향을 줌
- Partitioning:
  - 큰 데이터베이스를 파티션이라는 여러 서브셋으로 나눠 각각 다른 노드에 분산하는 방법
  - 6장에서 다룰 예정.
- 이러한 지식들을 바탕으로 어려운 트레이드 오프들에 대한 논의가 예정되어 있다.

---
# Ch.05 Replication

#### Questions From previous chapters
- (Chapter02 38pg에서 나온 질문) 관계형 vs 문서형 데이터 베이스 선택시 내결함성의 영향은?

---

##### Leaders and Followers
- 하나의 리더(Primary)가 모든 쓰기를 처리하고, 팔로워(Secondary)들은 리더의 복제 로그를 받아 자신의 데이터를 업데이트함. 
  - 읽기는 어느 노드에서든 가능함.
- 가장 전형적인 복제 모델
  - replica: DB 복제본을 가진 각 노드
  - leader: write 를 받는 노드
  - follower: leader 가 만든 변경사항을 따라가는 노드
- 왜 이런 구조를 쓰는가?
  - 여러 곳에서 동시에 write 를 받아 순서를 맞추는 문제를 피할 수 있음
  - “write ordering” 문제를 leader 1대로 몰아 단순화
- **Synchronous vs Asynchronous**:
  - 동기식 복제
    - leader 가 follower 의 ack 를 기다린 뒤 성공 처리
      - semi-synchronous 도 가능 
        - 최소 하나 follower 는 동기, 나머지는 비동기
    - 장점: follower 가 leader 와 거의 같은 상태를 유지
    - 단점: follower 가 느리거나 죽으면 write latency / availability 가 크게 나빠짐
  - 비동기식 복제
    - leader 가 follower 응답을 기다리지 않고 바로 성공 처리
    - 장점: 리더가 팔로워의 응답을 기다리지 않으므로 빠름
    - 단점: 리더 장애 시 복제되지 않은 최신 데이터가 유실될 수 있음.
- **Setting Up New Followers**
  - 진행 순서:
    1. leader 의 consistent snapshot 생성
    2. snapshot 을 새 follower 로 복사
       - 리더의 스냅샷이 리더의 복제 로그의 어디쯤인지에 대한 위치 정보 (MySQL의 binlog coordinates)을 스냅샷이 알고 있어야 함
    3. snapshot 이후의 변경 로그를 replay 하여 catch-up
  - snapshot 시점과 replication log 위치를 연결
    - 스냅샷 이후의 변경분은 이 복제 로그로 이어받음
- **Handling Node Outages**:
  - 팔로워 장애: 리더에서 받던 로그의 위치를 기억해두었다가 재연결 시 이어서 받음 (Catch-up).
  - 리더 장애 (Failover): 팔로워 중 하나를 새로운 리더로 승격시켜야 함. 타임아웃으로 장애를 판단하며, Split brain을 조심해야 하고, 기존 리더가 돌아오면 팔로워로 강등시켜야 함.
- **Implementation of Replication Logs**:
  - Statement-based: SQL 구문 자체를 복제. 
    - `NOW()`, `RAND()` 같은 비결정적 함수로 인해 복제본 간 데이터가 달라질 수 있어 위험함.
    - autoincrement 됙거나, 존재하는 다른 로우 데이터를 사용하는 경우에 모든 레플리카가 동시에 동작해야함 (id기반이 달라지지 않아야 하기에)
  - **WAL (Write-ahead log)**: 스토리지 엔진의 디스크 블록 변경 수준을 복제. 
    - 복제 속도가 빠름 그러나 스토리지 엔진 버전에 강하게 종속됨. -> 리더와 팔로워의 버전을 맞추는 것이 강제됨
  - **Logical (Row-based) log**: 스토리지 엔진과 분리된 형태의 로그. 
    - 버전 호환성이 좋고 외부 시스템(CDC)에서 파싱하기 좋음.

##### Problems with Replication Lag
- leader-follower 에서 follower 가 async 라면 복제 지연이 생김
- 결국 같은 시점에 leader 와 follower 를 조회하면 다른 값을 볼 수 있음
- 즉 eventual consistency 는 여기서 자연스럽게 나타남
  - follower 가 언젠가 따라잡겠지만, 지금 당장은 다를 수 있음
- 대부분 읽기 중심의 시스템에서는 다수의 비동기 팔로워를 두는 읽기 확장(Read-scaling) 아키텍처를 사용 ->Eventual Consistency로 인한 지연(Lag)이 발생
- **Reading Your Own Writes**: 
  - 사용자가 방금 write 했는데, 직후 read 를 아직 write되지 않은 follower 에서 하면 사용자 입장에서는 “내 저장이 안 된 것 같은데?” 라고 느낄 수 있음 
  - read-after-write consistency (read-your-writes) 가 보장 되어야함.
    - 유저가 수정한 내용을 즉시 볼 수 있게 보장해야 함
    - 보장할 수 있는 방법의 예시:
      - 사용자가 수정한 가능성이 있는 데이터는 leader 에서 읽기
        - 유저가 어플리케이션의 대부분을 수정 가능할 경우는 활용이 어려움
      - 마지막 write 이후 일정 시간은 leader 에서만 읽기
      - client 가 자신의 마지막 write timestamp/log position 을 기억하고, 그 이상 반영된 replica 에서만 읽기
        - 실제 시간을 쓰기보단 logical timestamp (쓰기 작업의 순서를 나타내는 값, 로그 시퀀스 넘버 등) 을 쓰는게 좋다 (자세한 내용은 8장에서)
      - 여러 데이터 센터에 레플리카가 존재한다면, 지연이 많이 발생하니 리더가 있는 곳으로 보내야함
- **Monotonic Reads**: 
  - 같은 사용자가 첫 번째 read 에서는 최신 replica 를 보고 두 번째 read 에서는 더 stale 한 replica 를 보면 시간이 되돌아간 것 처럼 느껴짐
    - 해당 유저의 요청은 항상 동일한 레플리카로 라우팅해서 해결!
  - monotonic reads 는 한 사용자가 연속적으로 읽을 때, 이전 read 보다 더 오래된 상태를 보지 않게 하는 보장
    - 강한 일관성 보다는 약하지만 최종적 일관성 보다는 강한 보장
    - 보장할 수 있는 방안:
      1. 사용자별로 같은 replica 에 sticky 하게 붙이기
         - 예: userId hash 기반으로 replica 선택
  - 이 보장은 “항상 최신”을 보장하진 않음 다만 “봤던 것보다 더 과거로 후퇴하는 경험”을 막음
- **Consistent Prefix Reads**: 
  - causality (인과성, 원인과 결과) 가 있는 write 들이 잘못된 순서로 보이는 문제
  - 이는 서로 다른 write 가 서로 다른 replica lag 때문에 뒤집혀 보이기 때문
    - 특히 partitioned system 에서는 전역 순서가 더 어려워짐
  - 인과성 있는 쓰기들을 같은 파티션으로 라우팅 으로 해결
- Solutions for Replication Lag
  - lag을 완전히 없애기보다 어디서는 허용하고 어디서는 안 허용할지 트레이드 오프
    - 중요한 질문:
      - 어떤 화면/기능은 stale read 허용 가능한가?
      - 어떤 기능은 반드시 read-after-write 가 필요한가?
      - causal order 가 깨지면 안 되는 데이터는 무엇인가?
  **- 복제 전략이 곧 제품 UX 정책**

##### Multi-Leader Replication
- single-leader 의 한계:
  - leader 1대가 write 병목이 됨
  - leader 와의 네트워크가 끊기면 write 불가
- multi-leader replication: 리더가 여러 개 존재하며, 각각의 리더가 쓰기를 받아 다른 리더 및 팔로워와 동기화함.
  - 여러 노드가 leader 역할을 하며 write 수락
  - 각 leader 가 다른 leader 에게 비동기로 변경을 전파
- **Use Cases for Multi-Leader Replication**:
  - multi-datacenter operation
    - write 요청에 대해 각 데이터센터에서 local write 처리 가능
    - 데이터 센터에서 데이터 센터간으로 동기화시의 latency 를 숨길 수 있음
    - 데이터센터 outage에 대한 내성
      - 리더가 죽어도 페일오버 되며 팔로워가 해당 데이터센터의 신규 리더가 될 수 있음
    - 네트워크 문제에 대한 내성
    - 기본으로 지원하는 디비도 있지만, 추가적인 툴과 함께 사용해야 하는 경우도 있다
      - MySQL - Tungsten Replicator, PostgreSQL - BDR, Oracle - GoldenGate
    - Write Conflict 이 존재할 수 있고, 이는 [handling-write-conflicts에서 다룰 예정.](#handling-write-conflicts)
  - Clients with offline operation
    - 잠시 인터넷이 끊기더라도 동작을 해야하는 경우 (몇 시간 혹은 몇일이 될 수도 있음)
    - 클라이언트가 offline 동안 local 에 write 누적 후 나중에 동기화
  - collaborative editing
    - ex: google doc, notion 등
    - 여러 사용자가 각자 local replica 에서 수정 후 합침

<a id="handling-write-conflicts"></a>
- **Handling Write Conflicts (충돌 처리)**:
  - single-leader 는 순서를 중앙에서 정해 conflict 를 크게 줄였음
  - multi-leader 는 그 전제를 버렸기 때문에 conflict resolution 이 핵심 과제가 됨
  - Write Conflicts: 동시에 같은 데이터를 다른 리더를 통해 수정할 경우 수정 사항을 합치는 과정에서 발생하는 충돌
    - ex: 두명의 유저가 동시에 게시글의 타이틀을 A 에서 B나 C 로 바꾸려할 때, 각각 다른 리더에서 이게 발생하게 된다면 두 요청을 성공할 것이나 이후 비동기 병합과정에서 문제가 발생함
  - 동기 vs 비동기적 감지
    - 비동기 감지: 쓰기 요청에 대한 처리가 빨라지나, 같은 대상에 대한 두개 이상의 쓰기 요청이 동시에 적용되었을 때 너무 늦게 감지 될 수 있음
    - 동기: 모든 레플리카 들에게 쓰기가 잘 되었다 확인을 받아 쓰기를 보장 -> 이러면 차라리 싱글 리더로 가야함..
  - 가능한 해결 방안:
    - Conflict avoidance
      - 같은 레코드에 대해 여러 리더에서 동시에 수정하지 않게 설계하여 충돌을 회피
        - 하지만 만약 데이터센터의 장애, 유저가 다른 데이터센터가 더 가까운 지역으로 이동하는 등의 상황이 벌어지면 결국 해당 레코드가 이전 리더에서 사용할 수 없게 됨으로 불가능해짐
    - Converging toward a consistent state
      - 모든 수정 사항들이 복제 (다른 리더 포함)에 전파되었을 때 최종적으로 같은 값을 가져야함.
      - 적용 가능한 방안:
        - 모든 쓰기 요청에 유니크 ID(타임스탬프, 랜덤난수, UUID, 해시값)를 부여하고, 가장 높은 값을 위너로 취급해 다른 모든 쓰기를 날리고 해당 쓰기만 적용.
          - 타임스탬프가 쓰일 경우 last write win 이라는 테크닉이 됨
          - 간단하지만 데이터 유실의 위험이 큼. 자세한 내용은 [Detecting Concurrent Writes에서 다룰 예정](#Detecting Concurrent Writes)
        - 모든 레플리카에 유니크 ID를 부여하고, 더 높은 ID를 가진 레플리카의 데이터를 우선하는 방식
          - 데이터 로스의 위험 존재
        - 충돌을 별도의 데이터로 저장후, 어플리케이션 코드에서 충돌 발생시 해결하도록 작성 (ex: 유저에게 메세지를 보여주는 등)
    - Custom Conflict resolution logic
      - 쓰기나 읽기 실행시 실행되는 어플리케이션 코드를 사용한 해결방안.
        - 쓰기:
          - 복제된 변경 사항 로그에서 충돌 감지 -> 충돌 핸들러 호출하여 백그라운드에서 실행
        - 읽기:
          - 모든 충돌 쓰기를 저장. 이후 읽기시 여러 버전의 데이터가 어플리케이션에 리턴되고 어플리케이션에서 충돌 내용을 해소해 다시 데이터 베이스에 기록.
    - what is conflict?
      - 회의실 예약의 경우, 동시에 같은 방을 같은 시간에 예약을 할 때 다른 리더에서 발생하게 되면 어쩔거냐? 같은 문제가 있음
      - [7장](chapter07.md), [12장](chapter12.md)에서 자세히 다룰 예정
- **Multi-Leader Replication Topologies**
  - 각각의 노드에서 다른 노드로 어떻게 쓰기가 전파되는지에 대한 기술
  - All-to-all
    - 모든 leader가 모든 leader에게 전파(가장 일반적)
  - Circular & Star: 한 노드가 한 방향으로 전달(MySQL 기본 토폴로지)
  - Star/Tree: root가 다른 노드들에게 전달(트리로 일반화 가능)

##### Leaderless Replication
- single/multi leader는 “클라이언트가 한 노드에 write → DB가 복제”
- leader가 write 순서를 결정하고 follower가 같은 순서로 적용
- **Leaderless**
  - 어떤 replica든 클라이언트 write를 직접 받는다(“leader” 개념 포기)
  - 클라이언트가 여러 replica에 직접 쓰거나(coordinator가 대신 보내기도 함),
  - coordinator가 있더라도 leader처럼 write ordering을 담당하지는 않음
- Amazon Dynamo로 다시 주목 → Riak/Cassandra/Voldemort 등이 Dynamo-style
- **Writing to the Database When a Node Is Down**
  - 기존 리더 베이스의 방식과 달리, **failover 가 필요없음**
  - 클라이언트가 3개의 레플리카에 병렬로 write 요청을 보냄 -> 2개 이상이 성공하면 해당 쓰기 요청을 성공 처리 ([Quorum reads/writes](#Quorum_reads_writes) 에서 다룰 예정)
    - 문제: 
      - 다운 노드 복귀 후 stale read
        - 다운 중 발생한 write가 해당 노드에 없으므로, 복귀 직후 그 노드에서 read하면 오래된 값이 나올 수 있음
    - 해결: 
      - read도 여러 노드에 병렬 전송
        - r개의 노드에 동시에 요청해서 응답 비교
        - version number로 최신/구버전을 판단(동시성/버전은 뒤에서 심화)
  - 결국 모든 replica가 따라잡게 하려면?
    - Dynamo-style에서 흔한 2가지 메커니즘
      - **Read repair**
        - 읽기 시 stale 응답을 감지하면, 클라이언트(또는 시스템)가 최신 값을 stale replica에 다시 써서 고침
        - 자주 읽히는 값에 특히 잘 동작
      - **Anti-entropy(백그라운드 동기화)**
        - 백그라운드 프로세스가 replica 간 차이를 계속 찾고 누락 데이터를 복사
        - leader-based의 replication log처럼 “순서대로” 복사하지 않으며, 지연이 클 수 있음
    - 주의: 어떤 시스템은 구현하지 않을 수도 있다(Voldemort는 anti-entropy가 없음)
      - anti-entropy가 없고 read repair만 있으면 “잘 안 읽히는 값”은 오래 누락될 수 있어 **내구성(durability)**이 떨어질 수 있음
  <a id="Quorum_reads_writes"></a>
  - Quorum reads/writes (n, w, r)
    - n: 해당 값이 저장되는 replica 수
    - w: write 성공으로 치기 위한 최소 ACK 수
    - r: read 시 최소로 응답을 받아야 하는 노드 수
    - `w + r > n`이면 “겹치는 노드가 있다” → 최신 값일 가능성이 높아짐(엄밀 보장은 뒤에서 깨짐)
    - 흔한 선택: n은 3/5 같은 홀수, `w=r=(n+1)/2`(올림) → “majority quorum”
    - 워크로드에 따라 조절 가능
      - 읽기 많고 쓰기 적으면 `w=n, r=1`로 읽기 빠르게 가능
      - 단, 노드 1개만 실패해도 모든 write가 실패할 수 있음(가용성 하락)
    - 클러스터 노드 수는 n보다 많을 수 있고,
    - 어떤 값은 전체 노드가 아니라 **n개 노드에만 저장**됨 → 큰 데이터셋을 위해 partitioning 필요([Ch6에서 다룰 예정](chapter06.md))
  - “w/r만큼만 기다린다”의 의미
    - 실제 요청은 보통 n개 replica에 병렬로 다 보냄
    - 성공/실패 판정은 “w개 또는 r개 성공 응답을 기다리느냐”로 결정
    - w 또는 r을 만족 못하면 에러
  - 노드가 unavailable함의 정의
    - 다운, 디스크 full, 네트워크 단절, 기타 에러 등 원인은 다양
    - 중요한 건 **성공 응답이 왔는지 여부**이며, 원인 구분은 필수가 아님
- **Limitations of Quorum Consistency**
  - ㅋ
----