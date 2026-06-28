# Ch.03 Page tables

----
- **page table 이란?**
  - OS가 각 프로세스에게 자기만의 private address space와 memory가 있는 것처럼 보이게 만드는 대표적인 메커니즘
  - virtual address가 어떤 physical memory를 가리키는지 결정하는 역할을 함
  - 서로 다른 프로세스의 주소공간을 격리(isolation)하고, 하나의 physical memory 위에 multiplexing 할 수 있게 해줌

## 3.1 Paging hardware
- RISC-V instruction은 user/kernel 여부와 상관없이 **virtual address** 를 사용함
  - 실제 RAM은 **physical address** 로 indexing 됨
  - page table hardware가 virtual address -> physical address 변환을 담당
- xv6는 RISC-V의 **Sv39** 모드를 사용
  - 64-bit virtual address 중 하위 39 bit만 사용
  - 상위 25 bit는 사용하지 않음
- Sv39에서 virtual address 구조:
  - 27 bit: page table index로 사용
  - 12 bit: page offset으로 사용
  - page 크기 = 4096 bytes = 2^12
- **PTE(Page Table Entry)**:
  - 44-bit PPN(physical page number)
  - permission / valid flag들
- 주소 변환의 큰 흐름:
  1. virtual address의 상위 27 bit로 page table에서 PTE를 찾음
  2. PTE의 PPN을 physical address의 상위 bit로 사용
  3. virtual address의 하위 12 bit offset은 그대로 physical address의 offset으로 복사

----
#### 3-level page table
- Figure 3.1은 flat array처럼 설명하지만, 실제 RISC-V page table은 **3-level tree** 구조 (Figure 3.2 참조)
- root page table page:
  - 4096 bytes
  - 512개의 PTE를 가짐
    - 4096 / 8 = 512
- virtual address의 27-bit index는 9 bit씩 3개로 나뉨
  - L2: root page directory에서 PTE 선택
  - L1: 중간 page directory에서 PTE 선택
  - L0: 마지막 PTE 선택
  - 나머지 12 bit는 page 내부 offset
- page table tree를 쓰는 이유:
  - virtual address space는 매우 크지만 실제로 mapping되는 영역은 작을 수 있음
  - 비어 있는 큰 virtual address range에 대해 하위 page directory를 아예 만들지 않아도 됨
  - flat page table보다 메모리를 훨씬 아낄 수 있음
- 단점:
  - 주소 변환 하나를 위해 최대 3개의 PTE를 memory에서 읽어야 함
    - 이 비용을 줄이기 위해 CPU는 **TLB(Translation Look-aside Buffer)** 에 page table entry를 cache함

----
#### PTE flags
- PTE는 mapping의 의미와 접근 권한을 flag로 표현함
  - `PTE_V`: valid bit. set되어 있지 않으면 해당 PTE는 유효하지 않고 page fault 발생
  - `PTE_R`: read 가능
  - `PTE_W`: write 가능
  - `PTE_X`: instruction으로 execute 가능
  - `PTE_U`: user mode에서 접근 가능
    - set되어 있지 않으면 supervisor mode에서만 사용 가능
- page fault가 나는 대표 상황:
  - 필요한 level의 PTE가 valid하지 않음
  - 권한에 맞지 않는 접근을 함
    - ex: write 권한 없는 text page에 store 시도

----
#### satp register
- CPU가 어떤 page table을 사용할지는 `satp` register가 결정함
  - kernel이 root page table page의 physical address를 `satp`에 기록
  - 이후 CPU가 실행하는 instruction의 address는 `satp`가 가리키는 page table 기준으로 변환됨
- 각 CPU는 자기 own `satp`를 가짐
  - 서로 다른 CPU가 서로 다른 process를 실행할 수 있음
  - 각 process는 private address space를 설명하는 자기 page table을 가짐

----
## 3.2 Kernel address space
- xv6는 시작할 때 kernel address space를 설명하는 **single kernel page table** 을 만든다
  - kernel이 physical memory와 hardware resource를 예측 가능한 virtual address로 접근하기 위함
  - layout 관련 상수는 `memlayout.h`에 선언됨

#### QEMU에서의 physical address layout
- `0x80000000` 아래쪽에는 RAM이 아니라 device register들이 있음
  - UART, VIRTIO disk, PLIC, CLINT 등
- 이런 device들은 memory-mapped I/O 방식으로 노출됨
  - kernel이 특정 physical address를 read/write하면 RAM 접근이 아니라 device와 통신하는 의미가 됨 => Shutdown 구현이 가능했던 이유

#### Direct mapping
- xv6 kernel은 대부분의 physical RAM과 device register를 **virtual address == physical address** 로 mapping함
  - 이를 **direct mapping**이라고 부름
  - kernel이 physical address `x`를 접근하고 싶으면 virtual address `x`를 load/store하면 됨
- kernel code 자체도 `KERNBASE = 0x80000000`에 위치
  - virtual address에서도 `0x80000000`
  - physical memory에서도 `0x80000000`
- direct mapping 덕분에 kernel 코드가 physical address를 포인터처럼 쉽게 사용할 수 있음
  - 예를 들어 allocator가 user memory용 physical page를 반환하면, kernel은 그 주소를 virtual address처럼 써서 copy 가능

----
#### direct mapping이 아닌 kernel virtual address
- xv6 kernel address space에는 direct mapping이 아닌 mapping도 있음

1. Trampoline page
   - virtual address space의 맨 위쪽에 mapping됨
   - user page table에도 같은 virtual address로 mapping됨
   - 하나의 physical page가 kernel virtual address space 안에서 두 번 보임
     - direct mapping 위치
     - virtual address space 최상단 trampoline 위치
   - trampoline의 역할은 [4장](chapter04.md)에서 user/kernel 전환 흐름과 함께 설명됨

2. Kernel stack pages
   - 각 process는 자기 kernel stack을 가짐
   - xv6는 kernel stack을 high kernel virtual address에 mapping함
   - stack 바로 아래에는 invalid PTE로 된 **guard page** 를 둠
     - kernel stack이 overflow되면 guard page 접근으로 page fault 발생
       - 이 경우 xv6는 panic으로 멈춤
   - guard page가 없다면 stack overflow가 다른 kernel memory를 조용히 덮어쓸 수 있음
     - 깨진 kernel state로 계속 실행하는 것보다 낫기에 panic 을 사용함
   - kernel stack의 physical page는 direct mapping 주소로도 접근 가능
     - 하지만 실제 stack 사용은 high-memory mapping을 통해 함
     - direct mapping만 사용하면 guard page를 만들기 위해 원래 접근 가능해야 하는 physical memory mapping 일부를 비워야 해서 불편해짐

## 3.3 Code: creating an address space
- xv6의 address space / page table 조작 코드는 대부분 `kernel/vm.c`에 있음
- 중심 타입:
  - `pagetable_t`
    - RISC-V root page-table page를 가리키는 포인터
      - kernel page table일 수도 있고 process별 user page table일 수도 있음
- 중심 함수:
  - `walk`
    - virtual address에 해당하는 PTE를 찾아줌
  - `mappages`
    - virtual address range에 physical address range를 mapping하는 PTE를 설치
- 페이지 테이블 조작 함수들:
  - `kvm...`: kernel page table 조작
  - `uvm...`: user page table 조작
  - 그 외 일부 함수는 kernel/user 양쪽에 공통 사용
- `copyin`, `copyout`:
  - system call argument로 넘어온 user virtual address에서 kernel로 copy하거나 kernel에서 user로 copy
    - user virtual address를 직접 physical address로 변환해야 하므로 `vm.c`에 있음

----
#### kernel page table 생성 흐름
- boot 초기에 `main`이 `kvminit` 호출
  - `kvminit`은 `kvmmake`를 통해 kernel page table 생성
  - 이 시점에는 paging이 아직 켜지지 않았기 때문에 address가 physical memory를 직접 가리킴
- `kvmmake`의 작업:
  1. root page-table page용 physical page 할당
  2. `kvmmap`으로 kernel이 필요한 translations 설치
     - translation이란?
       - kernel instructions, data, `PHYSTOP`까지의 physical memory, device register memory range
  3. `proc_mapstacks`로 각 process의 kernel stack mapping 설치

#### mappages
- `mappages`는 virtual address range를 page 단위로 돌며 mapping을 설치함
  - 각 virtual address마다 `walk`로 PTE 주소를 얻음
  - PTE에 physical page number와 permission을 기록
  - `PTE_V`를 set해 valid mapping으로 만듦

#### walk
- `walk`는 RISC-V page table hardware가 하는 일을 software로 흉내냄
  - virtual address의 L2/L1/L0 index를 이용해 page table tree를 내려감
  - 마지막 level의 PTE 주소를 반환
- 중간 level의 PTE가 valid하지 않은 경우:
  - `alloc`이 set되어 있으면 새 page-table page를 할당하고 PTE에 연결
  - `alloc`이 set되어 있지 않으면 실패
- 중요한 전제:
  - xv6 kernel은 physical memory를 direct mapping하고 있음
  - `walk`가 PTE에서 다음 level page table의 physical address를 꺼낸 뒤, 그 주소를 virtual address처럼 사용 가능함

----
#### kvminithart와 TLB flush
- 각 CPU는 `main`에서 `kvminithart`를 호출해 kernel page table을 실제 CPU에 설치함
  - root page table의 physical address를 `satp`에 기록
- 이 순간부터 CPU는 kernel page table을 사용해 address translation을 수행
- kernel이 계속 정상 실행되는 이유:
  - kernel page table이 direct-mapped라서 paging 전후로 kernel address가 같은 physical location을 가리킴
- TLB 문제:
  - CPU는 page table entry를 TLB에 cache함
  - page table이 바뀌었는데 TLB가 오래된 mapping을 계속 쓰면 격리가 깨질 수 있음
    - 예: 이미 다른 process에게 할당된 physical page를 예전 mapping으로 접근
- RISC-V는 `sfence.vma` instruction으로 현재 CPU의 TLB를 flush함
  - xv6는 `satp` reload 후 `kvminithart`에서 실행
  - user/kernel 전환 trampoline 코드의 `uservec`, `userret`에서도 실행
- `satp`를 바꾸기 전에도 `sfence.vma`가 필요함
  - 이전 load/store와 page table update가 완료되었음을 보장
  - 이전 memory access는 old page table 기준, 이후 access는 new page table 기준으로 정리되게 함

## 3.4 Physical memory allocation

- kernel은 run-time에 physical memory를 할당/해제해야 함
  - page table
  - user memory
  - kernel stack
  - pipe buffer 등
- xv6는 kernel 끝부터 `PHYSTOP`까지의 physical memory를 run-time allocation 대상으로 사용
- allocation 단위:
  - 항상 4096-byte page 단위
  - 작은 object allocator는 없음
- free page 추적 방식:
  - free page들 자체를 linked list node로 사용
  - allocation = free list에서 page 하나 제거
  - free = page를 free list 앞에 추가

## 3.5 Code: Physical memory allocator

- allocator 코드는 `kernel/kalloc.c`에 있음
- 핵심 자료구조:
  - free list
  - 각 free page의 시작 부분에 `struct run`을 저장
  - `struct run`은 다음 free page를 가리키는 pointer를 가짐
- 왜 free page 내부에 metadata를 저장하나?
  - free page는 어차피 다른 용도로 사용 중이 아니기 때문
  - 별도 metadata memory를 둘 필요가 없음
- free list는 spin lock으로 보호됨
  - 이 lock은 Chapter 7에서 자세히 다룸

----
#### 초기화 흐름
- `main`이 `kinit` 호출
- `kinit`은 kernel 끝부터 `PHYSTOP`까지의 page를 free list에 넣음
  - xv6는 실제 physical memory 크기를 hardware config에서 읽지 않고, 128MB RAM이 있다고 가정
- `freerange`가 page 단위로 `kfree`를 호출해 free list를 채움
  - PTE가 가리킬 수 있는 physical address는 4096-byte aligned여야 함
  - 그래서 `PGROUNDUP`으로 시작 주소를 page boundary에 맞춤

#### kfree
- `kfree`는 해제되는 page의 모든 byte를 `1`로 채움
  - dangling reference를 빨리 터뜨리기 위함
  - free 후에도 예전 내용이 남아 있으면 bug가 조용히 지나갈 수 있음
- 그 후 page를 `struct run *`으로 casting해서 free list 앞에 붙임

#### kalloc
- `kalloc`은 free list의 첫 page를 떼어 반환
- allocator 코드에 type cast가 많은 이유:
  - address를 integer처럼 다루며 page 단위 산술을 하기도 하고
  - pointer처럼 다루며 해당 memory에 `struct run`을 쓰기도 하기 때문

## 3.6 Process address space

- 각 process는 자기 page table을 가짐
  - xv6가 process를 switch할 때 page table도 함께 바꿈
- process user address space는 0에서 시작해 이론상 `MAXVA`까지 가능
  - 실제로는 그중 작은 일부만 physical memory에 mapping됨

#### user address space 구성
- program text
  - instruction 영역
  - `PTE_R`, `PTE_X`, `PTE_U`
  - write 권한 없음
- program data
  - initialized data / global data 등
  - `PTE_R`, `PTE_W`, `PTE_U`
  - execute 권한 없음
- stack
  - `PTE_R`, `PTE_W`, `PTE_U`
- heap
  - `PTE_R`, `PTE_W`, `PTE_U`

----
#### permission을 세밀하게 나누는 이유
- user process 자체를 더 단단하게 만들기 위함
- text page에 `PTE_W`가 없으면:
  - process가 실수로 자기 instruction을 수정할 수 없음
  - 예: null pointer write로 address 0에 store하려고 하면 page fault 발생
- data page에 `PTE_X`가 없으면:
  - data 영역으로 jump해서 실행하는 실수를 막을 수 있음
- 보안 관점에서도 중요
  - 공격자는 program bug를 exploit으로 바꾸려 할 수 있음
  - page permission, address space layout randomization 같은 기법이 exploit을 어렵게 만듦

#### user stack
- xv6의 user stack은 기본적으로 1 page
- `exec`가 만든 초기 stack에는 다음이 들어 있음
  - command-line argument string들
  - 그 string을 가리키는 pointer array
  - `main(argc, argv)`가 호출된 것처럼 보이게 하는 값들
- stack 바로 아래에는 guard page가 있음
  - xv6는 `PTE_U`를 clear해서 user mode에서 접근 불가능하게 만듦
  - stack overflow로 아래쪽 주소를 사용하면 page fault 발생
- 실제 OS라면 stack overflow 시 자동으로 stack memory를 늘릴 수도 있음
  - xv6는 단순하게 fault 처리

----
#### process page table이 보여주는 page table의 장점
- 서로 다른 process의 같은 virtual address가 서로 다른 physical page로 mapping될 수 있음
  - process마다 private memory 제공
- process 입장에서는 memory가 virtual address 0부터 contiguous하게 보임
  - 실제 physical memory는 non-contiguous여도 됨
- trampoline page처럼 하나의 physical page를 모든 process address space에 같은 virtual address로 mapping할 수 있음
  - 단, `PTE_U` 없이 mapping해서 user mode에서는 접근 불가
  - kernel 전환 코드만 사용할 수 있음

## 3.7 Code: exec

- `exec` system call은 현재 process의 user address space를 executable file의 내용으로 교체함
  - 기존 program을 버리고 새 program image로 바꿈
- kernel 내부 구현은 `kexec`
- executable file은 보통 compiler/linker output이며 xv6는 ELF format을 사용함

#### ELF 로딩 흐름
- `kexec`의 큰 흐름:
  1. `namei`로 실행할 binary path를 열고 inode를 얻음
  2. ELF header를 읽음
  3. ELF magic number를 확인
     - `0x7F`, `E`, `L`, `F`
  4. 새 user mapping이 없는 page table을 `proc_pagetable`로 만듦
  5. 각 program segment에 대해:
     - `uvmalloc`으로 필요한 user memory 할당
     - `loadseg`로 file 내용을 memory에 load
  6. user stack을 만들고 argument를 stack top부터 복사
  7. 모든 준비가 성공하면 새 page table로 commit하고 old image 해제
- ELF program header는 각 segment가 memory 어디에 load되어야 하는지 설명함
  - text segment: 보통 virtual address 0에 load, read/execute 가능, write 불가
  - data segment: page boundary에 load, read/write 가능, execute 불가
- `filesz < memsz`인 segment도 있음
  - file에는 일부만 있고 나머지는 zero로 채워야 하는 영역
  - C global variable의 zero-initialized 영역이 대표 예시

----
#### loadseg와 walkaddr
- `loadseg`는 ELF segment를 page 단위로 memory에 읽어 넣음
- user virtual address에 해당하는 physical address를 찾기 위해 `walkaddr` 사용
  - xv6는 file 내용을 kernel buffer로 읽는 게 아니라, 최종 user memory에 직접 채워야 함
  - 따라서 user page table 기준의 address translation이 필요

#### argument stack 구성
- `kexec`는 user stack을 1 page 할당
- argument string들을 stack top에서 아래 방향으로 하나씩 복사
- 각 argument string의 user virtual address를 `ustack` 배열에 기록
- `argv` 끝에는 null pointer를 둠
- `main(argc, argv)` 전달 방식:
  - `argc`: system call return value 경로를 통해 `a0`에 들어감
  - `argv`: process trapframe의 `a1`에 들어감
- stack 아래 guard page는 argument가 너무 클 때도 도움이 됨
  - `copyout`이 inaccessible page로 복사하려 하면 실패하고 `-1` 반환

----
#### exec의 commit 시점
- `exec`는 중간에 실패할 수 있음
  - invalid program segment
  - memory allocation failure
  - file read failure 등
- 그래서 새 memory image를 준비하는 동안 기존 image를 바로 버리면 안 됨
  - 실패 시 old image로 돌아가 `exec` system call이 `-1`을 반환해야 하기 때문
- xv6는 새 image 준비가 완료된 뒤에만 commit함
  - 새 page table을 process에 연결
  - 그 후 old page table / old memory 해제

#### exec의 보안 위험
- ELF file 안의 virtual address는 user/process가 마음대로 조작할 수 있음
  - kernel이 검증 없이 믿으면 kernel memory를 덮거나 isolation을 깨는 exploit이 될 수 있음
- xv6는 여러 check를 수행함
  - 예: `ph.vaddr + ph.memsz`가 64-bit overflow를 일으키는지 검사
- 특히 예전 설계처럼 user address space 안에 kernel mapping도 같이 있었다면 더 위험함
  - malicious ELF가 kernel memory에 대응하는 address를 골라 file 내용을 kernel 쪽에 써버릴 수 있음
- RISC-V xv6에서는 kernel이 별도 page table을 가지므로 이 특정 위험은 줄어듦
  - `loadseg`는 process page table에 load하고 kernel page table에 load하지 않음
- 그래도 핵심 교훈:
  - kernel은 user가 제공한 binary/header/address/size를 절대 그대로 믿으면 안 됨
  - real-world kernel들도 이런 검증 누락으로 privilege escalation 취약점이 많이 생겼음

## 3.8 Real world

- 대부분의 OS는 xv6처럼 paging hardware를 memory protection과 mapping에 사용함
- 다만 real-world OS는 xv6보다 훨씬 복잡하게 paging과 page fault를 조합함
  - lazy allocation
  - demand paging
  - copy-on-write
  - memory-mapped files 등

#### xv6의 단순화
- xv6는 kernel virtual address와 physical address를 direct map함
- xv6는 RAM이 `0x80000000`에 존재하고 kernel도 그 주소에 load된다고 가정
  - QEMU에서는 동작함
  - 실제 hardware에서는 좋지 않은 가정
- real hardware에서는 RAM/device physical address layout이 예측 불가능할 수 있음
  - `0x80000000`에 RAM이 없을 수도 있음
- 실제 kernel은 page table을 이용해 제각각인 hardware physical layout을 kernel이 쓰기 좋은 predictable virtual layout으로 바꿈

#### xv6가 사용하지 않는 기능들
- RISC-V는 physical address level의 protection도 지원하지만 xv6는 사용하지 않음
- super pages
  - 큰 page를 사용해 page table overhead와 TLB overhead를 줄일 수 있음
  - memory가 많은 machine에서는 유리할 수 있음
  - 하지만 작은 program에 큰 page를 주면 memory 낭비가 큼
- ASID(Address Space Identifier)
  - page table 변경 시 전체 TLB를 flush하지 않고 특정 address space의 TLB entry만 flush할 수 있게 함
  - xv6는 사용하지 않음
- small object allocator
  - xv6 kernel allocator는 4096-byte page 단위만 제공
  - 실제 kernel은 다양한 크기의 작은 object를 효율적으로 할당하는 allocator가 필요함

#### memory allocation의 현실적 난점
- memory allocator의 오래된 핵심 문제:
  - 제한된 memory를 효율적으로 사용
  - 미래에 어떤 크기/패턴의 allocation request가 올지 모르는 상태에서 준비
- 현대 시스템에서는 space efficiency뿐 아니라 speed도 매우 중요함