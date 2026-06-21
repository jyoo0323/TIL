# Ch.02 Operating system organization

----
- OS의 핵심 요구사항: 여러 활동 (activities) 를 동시에 지원하는 것
  - CPU, 메모리 등을 time share 할 수 있어야만 함
  - 프로세스간 격리(isolation)가 존재해 malfunction 하는 프로세스 하나가 다른 프로세스에 영향을 주지 않아야함
    - 그러나 완전한 격리 (Complete Isolation)은 너무 과하다 한다 (프로세스간 interact 하는 경우가 있을 수 있음 ex: pipelines)
  - OS가 충족해야할 3가지 조건:
    - Multiplexing
    - Isolation
    - Interaction

## 2.1 Abstracting physical resources
- 운영 체제가 왜 있어야 할까? 시스템 콜을 전부 어플리케이션에서 자유롭게 사용하는게 어플리케이션 입장에서 더 효율적이지 않나?
  - 이 접근 방식의 가장 큰 문제는, CPU 같은 리소스를 각 어플리케이션이 다른 어플리케이션을 위해 잠시 넘겨주도록 만들어져야 한다는 점이다.
    - 이렇게 만드는건 가능은 하다. 근데 버그가 발생했을때는 문제가 될 수 있다.
      - 그리고 프로그램은 버그가 발생할일 확률이 안발생할 확률보다 높다
- 위의 문제 상황을 막기위해 OS는 어플리케이션이 직접 **민감한 하드웨어 리소스에 접근하는 것을 금지(forbid)하고 리소스를 서비스로 추상화한다**
- 또 OS는 어플리케이션들이 time share 하고 있다는 것을 모르게 CPU들을 프로세스간에 투명하게 전환(switch)해 레지스터 상태를 필요에 따라 저장/복구(restore)함
  - 이 성질로 일부 애플리케이션이 무한 루프에 있더라도 OS가 CPU를 공유할 수 있다

## 2.2 User mode, supervisor mode, and system calls
- 강한 격리를 위해선 애플리케이션과 OS 간에 확고한 경계가 요구됨.
  - 어플리케이션은 설령 버그가 있거나 악성 어플리케이션이여도 다른 프로그램이나 OS의 operation을 방해할 수 없어야 한다.
  - OS는 어플리케이션이 OS의 데이터 구조/instruction 를 수정할 수 없어야 하고 다른 프로세스의 메모리에 접근할 수 없어야 함
- 격리를 위한 CPU 하드웨어 지원: 3가지 권한 레벨
  - 머신 모드
    - 최상위 권한 
    - 부팅 시점에 컴퓨터 설정을 위해 사용됨
  - 슈퍼바이저 모드
    - 커널의 레벨
    - 특권 명령 (pivileged instruction) 사용 가능
      - interrupt 를 활성/비활성화, 페이지 테이블의 주소를 들고있는 레지스터에서 읽기/쓰기 등의 권한이 있음
      - 하위 레벨 (유저 모드)의 프로세스에서 특권 명령 실행시 슈퍼바이저 모드로 전환되며 해당 어플리케이션을 종료시킴 
    - kernal space에서 동작함
  - 유저 모드
    - 유저가 실행시키는 프로세스의 레벨
    - user space 에서 동작
    - 커널 함수들을 직접 접근할 수 없음
      - 시스템 콜을 통해 커널과 상호작용(interact)
      - 시스템 콜을 통해 해당 CPU를 수퍼바이저 모드로 변경시키고 요청에 대한 검증 후 요청을 수행하거나 거절함

## 2.3 Kernel organization
- 핵심은 OS의 어떤 부분이 수퍼바이저 모드에서 동작해야하는가 이다.
-----
#### Monolithic Kernal:
- 모든 OS 기능을 하나의 커널에서 다루고 모든 시스템 콜이 수퍼바이저 모드에서 동작하는 커널
- 장점:
  - 수퍼바이저 모드를 사용하는 코드와 안하는 코드를 구분할 필요가 없음
  - OS의 여러 파츠가 협동하기 좋음 -> 하나의 프로그램이니까!
- 단점:
  - 커널이 방대&복잡해지는 경향이 있음
    - 한명의 개발자가 모든 파츠들의 상호작용을 다 이해하기 불가능해짐 -> 버그의 원인!
      - 커널에서의 버그는 컴퓨터 전체의 보안 공격 등에 취약해질 수 있어 치명적임
  - 많은 유닉스 커널은 모놀리식
    - window system 등은 유저 레벨 서버로 돌아간다함
    - 커널의 하위 시스템이 타이트하게 통합되어 높은 퍼포먼스를 냄
    - 우리가 배우는 xv6도 모놀리식
#### Microkernal
- 버그를 줄이는데 초점을 둔 커널로, 수퍼바이저 모드에서 돌아가는 코드를 최소화해 커널의 functionality 를 최소화한 커널로 수정을 위한 이해&분석하기 쉽다
- 그래서 OS의 대부분이 user 레벨 서버 프로세스로 동작한다.
  - ex: 파일 서버를 유저 모드로 실행한다 (수퍼바이저 모드 X)
  - 어플리케이션들이 이런 유저 모드로 실행된 서버들과 상호작용(interact)하기 위해 마이크로 커널은 메세지를 유저 레벨 프로세스간에 전달해주는 프로세스간 소통 메커니즘을 제공한다.
- 대부분의 기능이 유저 레벨 서버로 제공되기에 상대적으로 커널 심플하다
- Minix, L4, QNX 등의 OS가 마이크로커널을 활용
  - 주로 embedded 환경에서는 많이 사용된다 함
- 퍼포먼스 이슈로 마이크로커널임에도 유저 레벨 서비스들을 커널 스페이스에서 실행하는 OS들이 있다
  - 그러면 유저 스페이스에 있는 유저 레벨 서버들이 실제 리소스에 접근할 때는 다시 커널에 요청을 보내서 받는 방식인가?

----
## 2.5 Process overview
- xv6(및 Unix)에서 **격리의 단위 = process**
  - 프로세스 추상화가 막아주는 것:
    - 한 프로세스가 다른 프로세스의 메모리/CPU/파일디스키립터 등을 망가뜨리거나 훔쳐보는 것
    - 프로세스가 커널을 망가뜨리는 것 (커널의 격리 메커니즘을 전복 방지)
  - 버그/악의적 앱이 커널이나 하드웨어를 속일 수 있기에 프로세스 추상화는 신중하게 커널에 구현해야 함
  - 프로세스 구현에 쓰이는 메커니즘: user/supervisor mode flag, address space, thread의 time-slicing

- 격리를 위해 프로세스에 "자기만의 private machine을 가진 것 같은 환상(illusion)"을 줌:
  - **address space**: 자기만의 private memory 시스템처럼 보임 (다른 프로세스가 읽기/쓰기 불가)
  - 자기만의 CPU로 명령을 실행하는 것처럼 보임

#### Address space
- xv6는 **page table**(하드웨어:CPU의 레지스터가 구현)을 사용해 각 프로세스에 자기만의 주소공간을 줌
- page table이 가상주소(virtual address, RISC-V 명령이 다루는 주소)를 물리주소(physical address, CPU가 메모리에 보내는 실제 주소)로 변환(map)함
- 프로세스마다 별도의 page table이 있어 각자의 주소공간 정의
- 주소공간 레이아웃 (pg26 Figure 2.3, virtual address 0부터 시작):
  - 맨 위(MAXVA 근처)에 **trapframe**, **trampoline** 페이지
  - heap (malloc용, 프로세스가 위로 확장 가능)
  - user stack
  - user text and data (명령 -> 전역변수)
- 주소공간 최대 크기 제한:
  - RISC-V 포인터는 64비트지만, 하드웨어는 page table 조회시 하위 39비트만 사용
  - xv6는 그중 38비트만 사용 -> 최대 주소(**MAXVA**) = 2^38 - 1
- 주소공간 맨 위의 두 페이지:
  - **trampoline**(4096바이트): 커널로 진입/복귀하는 코드를 담음
  - **trapframe**: 커널이 프로세스의 user 레지스터를 저장하는 곳
  - 둘 다 커널 진입/탈출 전환에 쓰임 ([4장](chapter04.md)에서 설명)
----
#### Thread
- 커널이 각 프로세스 상태를 `struct proc`에 모아둠
  - 가장 중요한 것: page table, kernel stack, run state
  - `p->xxx` 표기로 멤버 참조 (ex: `p->pagetable`)
- 각 프로세스는 실행을 담당하는 **thread(스레드)** 를 가짐
  - thread는 CPU에서 실행 중이거나 / suspended(중단, 나중에 재개 가능) 상태일 수 있음
  - CPU를 프로세스간 전환할 수 있음 
    - 현재 thread를 중단하고 상태 저장 -> 다른 thread 상태 복원
  - thread 상태 상당수(지역변수, 함수 호출 리턴 주소)는 thread의 stack에 저장됨
- 각 프로세스는 두 개의 stack을 가짐: user stack & kernel stack
  - user 명령 실행 중: user stack만 사용, kernel stack은 비어있음
  - 커널 진입(system call/인터럽트) 중: 코드가 kernel stack에서 실행됨. user stack은 데이터는 남아있되 미사용
  - **kernel stack이 분리/보호되는 이유**: 프로세스가 user stack을 망가뜨려도 커널은 정상 실행 가능해야 하므로
- system call의 흐름:
  - user 코드가 `ecall` -> supervisor mode 전환 + PC를 커널 entry point로
  - entry point 코드가 kernel stack으로 전환 + system call 구현 실행
  - 완료되면 `sret` 명령으로 user mode 복귀 + `ecall` 직후부터 user 명령 재개
  - 프로세스의 thread는 I/O를 기다리며 커널에서 "block"할 수 있고, I/O 끝나면 멈춘 지점부터 재개
- `struct proc` 의 주요 멤버:
  - `p->state`: 프로세스가 allocated / ready to run / running / waiting for I/O / exiting 중 무엇인지
  - `p->pagetable`: RISC-V 하드웨어가 기대하는 형식의 page table을 담음. user space 실행시 이걸 사용
---
- process = address space(자기만의 메모리 영역이 있다는 환상) + thread(자기만의 CPU가 있다는 환상)
  - xv6에선 1 process = 1 address space + 1 thread
  - 실제 OS는 멀티 CPU 활용을 위해 한 프로세스에 여러 thread를 두는게 일반적

## 2.6 Code: starting xv6, the first process and system call

- xv6가 부팅해서 첫 프로세스를 실행하는 흐름 (`entry.S`, `start.c`, `main.c`, `init.c`):
  1. RISC-V 전원을 킴 
  2. 자체 초기화 + ROM의 **boot loader** 실행
  3. boot loader가 xv6 커널을 물리 주소 **0x80000000** 에 로드
    - 왜 0x0이 아니라 0x80000000? -> 그 이전 범위는 I/O 디바이스용
  4. boot loader가 `_entry` (entry.S)를 실행
    - 이 시점엔 paging 하드웨어가 비활성 상태 -> virtual address = physical address 직결
    - `_entry`는 C 코드를 돌릴 수 있게 stack(`stack0`) 을 셋업 (sp = `stack0 + 4096`, RISC-V stack은 아래로 자람)
  5. `_entry`가 C 코드 `start` (start.c) 호출
    - `start`는 machine mode에서만 가능한 셋업 수행 -> 특히 타이머 인터럽트용 clock chip 프로그래밍
    - `mret` 명령으로 supervisor mode 전환 + `main`으로 
      - 이를 위해 previous privilege를 supervisor로(`mstatus`), 리턴 주소를 `main`으로(`mepc`)
      - supervisor의 가상주소 변환 끔(`satp`에 0), 인터럽트/예외를 supervisor에 위임
  6. `main` (main.c)이 여러 디바이스/서브시스템 초기화 후 `userinit` 호출 -> 첫 프로세스 생성
    - 새로 생성된 프로세스는 모두 커널에서 `forkret`부터 실행 시작
    - 첫 프로세스는 특별히 `forkret`이 `kexec`를 불러 user 프로그램 **`/init`** 을 로드
  7. `forkret`이 `/init` 프로세스로 user space 복귀
    - **`init`** (init.c)이 필요시 새 console 디바이스 파일 생성 -> 그걸 fd 0, 1, 2로 open -> console에서 **shell 시작**
  8. 시스템 가동 완료

## 2.7 Security Model

- OS는 버그 있거나 악의적인 코드를 어떻게 다루나? -> 악의(malice)를 막는 게 우연한 버그보다 더 어려우므로 보안은 주로 악의 대응에 초점
- user 코드는 신뢰 0
  - 커널은 user-level 코드가 커널/타 프로세스를 망가뜨리려 최선을 다할 것이라 가정해야 함
    - 허용 범위 밖 포인터 역참조, user용이 아닌 명령 실행 시도, RISC-V control 레지스터 read/write 시도, device hardware 접근 시도, system call에 교묘한 값을 넘겨 커널을 속이려는 시도 등
  - 커널의 목표: 각 user 프로세스가 자기 user memory만 접근, 32개 범용 레지스터만 사용, system call이 의도한 방식으로만 커널/타 프로세스에 영향을 주도록 제한. 그 외 모든 동작은 차단 (대부분의 경우 절대적 요구사항)
- kernel 코드만 신뢰함
  - 커널 코드는 선의의 신중한 프로그래머가 작성 + 버그 없음 + 악의 없음으로 가정
    - ex: 내부 함수(spin lock 등)를 커널이 잘못 쓰면 큰 문제 -> 그러나 커널은 자기 함수를 올바르게 쓴다고 가정
  - 하드웨어 레벨에서도 RISC-V CPU/RAM/disk 등이 문서대로, 하드웨어 버그 없이 동작한다고 가정
- 현실은 단순하지 않음:
  - 악성 프로그램이 커널 보호 자원(disk space, CPU time, process table slot 등)을 소진해 시스템을 못쓰게 만드는 호출을 막기 어려움
  - 100% 버그 없는 커널/하드웨어는 사실상 불가능 -> 공격자가 알려진 커널/하드웨어 버그를 악용
    - 성숙하고 널리 쓰이는 Linux조차 새 취약점이 계속 발견됨
  - user/kernel 코드 경계가 흐려지기도 함
    - 일부 특권 user-level 프로세스가 핵심 서비스를 제공하기도 하고, 일부 OS는 특권 user 코드가 커널에 새 코드를 삽입 가능 (ex: Linux의 loadable kernel module, eBPF)
- 커널 버그에 대한 부분적 방어 = `panic()`
  - xv6는 불일치/복구불가 오류를 검사하다 발견하면 `panic()` 호출 -> 에러 메시지 출력 + 시스템 정지(halt)
  - panic은 바람직하진 않지만, 잘못된 커널 데이터/불법 동작(없는 메모리 참조 등)으로 인해 일관성 깨진 상태로 계속 도는 것보다 멈추는 게 안전
  - 커널 개발자는 panic을 보고 근본 버그를 찾아 고침

## 2.8 Real world
- 대부분의 OS가 process 개념을 채택했고, 대개 xv6의 것과 비슷함
- 다만 현대 OS는 한 프로세스 안에 여러 thread를 지원 -> 단일 프로세스가 멀티 CPU를 활용하게
  - 멀티스레드 지원엔 xv6에 없는 상당한 machinery가 필요
  - 인터페이스 변화도 동반됨 -> ex: Linux의 **`clone`**(fork의 변종) -> 프로세스 thread들이 어떤 자원을 공유할지 제어

## Exercises
- 첫번째 어프로치: OS에 할당된 최대 메모리에서 0 ~ MAXVA 까지 loop 돌면서 할당이 안된 페이지들이 있으면 그 페이지 수 * PGSIZE를 빼버리자!
```c
// sysfile.c에 추가한 시스템콜 메서드
uint64
sys_freemem(void)
{
  struct proc *p = myproc();
  uint64 va;
  uint64 usedpages = 0;
  pte_t *pte;

  for (va = 0; va < MAXVA; va += PGSIZE) {
    pte = walk(p->pagetable, va, 0);
    if (pte != 0 && (*pte & PTE_V)) // 매핑된 페이지 = 사용 중
      usedpages++;
  }

  // free = OS에 할당된 최대 물리 메모리 - 사용 중인 메모리
  return (PHYSTOP - KERNBASE) - usedpages * PGSIZE;
}
```
- 이 방식의 문제점: **이 코드의 동작은 실제 메모리가 아니라 가상 메모리를 조회해 비어있는 가상 메모리 공간을 카운트하는 방식임**
  1) 커널이 쓰는 메모리가 누락 - 커널 스택, 파이프 버퍼 등도 물리 페이지를 먹지만 유저 페이지 테이블엔 안 보이기에 사용중인데 감지가 안되는 문제가 있음
  2) 페이지 테이블 자신이 쓰는 페이지가 누락됨 -> 쓰이고 있는데도 누락되는 문제가 있음
  3) 공유 페이지가 중복 계산
- 그러면 어떻게 실제 물리 메모리에 접근할 수 있을까? 를 고민하며 다른 예제들과 비교해보니 대부분은 kalloc.c의 freelist 를 이용해 비어있는 물리 테이블에 접근하고, 그 테이블들을 계산하는 방식으로 계산했다.

```c
//kalloc.c 에 추가한 메서드
uint64
freemem(void)
{
  struct run *r;
  uint64 pages = 0;

  acquire(&kmem.lock);
  for (r = kmem.freelist; r != 0; r = r->next)
    pages++;
  release(&kmem.lock);

  return pages * PGSIZE;
}

.....
//sysfile.c 변경점
uint64
sys_freemem(void)
{
  return freemem();
}

.....

//새로 추가한 user 프로그램 freememcheck.c

#include "kernel/types.h"
#include "user/user.h"

int main() {
  printf("freememcheck has executed. It takes quite a bit a whiile to get the memery calculated..\n");
  uint64 result = freemem();
  printf("Currently free memory %lu \n", result);
  exit(0);
}
```