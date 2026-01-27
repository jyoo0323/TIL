# Ch.1 Reliable, Scalable, and Maintainable Applications  

## What is expected to be learned from this chapter?
- Reliability 의 정의와, 이를 발생시킬 수 있는 하드웨어, 소프트웨어, 휴먼 에러 등
- Scalability 의 정의와, Load, performance 의 정의 그리고 이 로드를 처리해 확장성을 키우는 것의 의미
- Maintainability 의 의미와, 이와 연관된 3가지 개념
- figure 1-1 같은 설계를 해야하는 이유와 목적, 그리고 해당 설계의 장단점
- 2012년 당시 트위터의 데이터 파이프라인이 왜 효율적이었는지, 어떤 개선이 필요한지
- 
## Questions to be answered as I read the book
- figure 1-1 의 Full-Text index는 뭘까?
- 하드웨어의 문제를 어떻게 소프트 웨어 개발자가 핸들하려하는 걸까?
- 확장성과 load 의 의미가 요청일까? 데이터의 양일까?
- 퍼포먼스에 대한 설명중 리스폰스 타임에 대한 일러스트레이션이 나온 이유는?
- Percentile 의 중요성은 무엇이며, 왜 이걸 퍼포먼스 쪽에서 길게 설명을 하며 강조하는가?
- Functional Requirements VS Nonfunctional Requirements?
- 챕터 후반부 summary애서 반복적으로 여러 다른 어플리케이션에서 나올거라고 하는 패턴과 테크닉은 무엇일까?
- data-intensive 와 compute-intensive의 차이?

----

### Thinking About Data Systems
- 데이터 스토어등 (디비, 메세지 큐 등)은 굉장히 비슷한 면이 있음에도 굉장히 다른 액세스 패턴을 가지고, 액세스 패턴이 다르기에 **성능적 특성이 다르고, 구현방식도 다름**
- 메세지 큐의 역할을 할 수 있는 레디스, 디비-라잌 durability 를 가진 카프카 등이 있다. 각 카테고리간의 차이가 희미해져간다.
- 최근에는 하나의 특성을 가진 툴로는 부족한 경우가 많기에 어플리케이션 코드에서 여러 툴을 사용함 (Figure 1-1)

### Reliability
- 소프트웨어에서 말하는 안정성: _"뭔가 잘못된 일이 생기더라도 계속 정상적으로 돌아감"_
  - 어플리케이션이 유저의 예상대로 동작함
  - 유저의 실수나, 예상되지 않은 사용 (버그, 어뷰징) 방법으로 사용해도 버팀
  - 예측한 로드와 데이터 양에 필요한 유스케이스에 맞는 퍼포펀스를 가짐
  - 어뷰징/비인증 엑세스를 방
- Faults (결함)과 Failure (장애)의 차이:
  - 결함: 하나의 컴포넌트가 정해진 스펙대로 동작하지 않음
  - 장애: 시스템 전체가 유저에게 서비스할 수 없음
- 결함을 완전히 없애는 것은 불가능함. -> 결함에 대한 내성 (tolerance) 를 가지는 것이 좋음
- 보안 문제의 경우는 돌이킬 수 없기 때문에 결함 내성을 가지기 보단 결함을 방지해야한다.

#### Hardware Faults
- 큰 데이터 센터등지에서는 하드웨어 이슈 (네트워크 선 뽑힘 등) 이 매우 많이 발생한다는 듯 하다
- 