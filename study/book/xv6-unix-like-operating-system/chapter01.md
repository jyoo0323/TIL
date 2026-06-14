# Ch.01 Operating System Interfaces

----
- OS의 역할:
  - 하나의 컴퓨터를 여러 프로그램이 공유(share) 하게 해줌
  - low-level 하드웨어를 관리하고 추상화(abstract) 함
    - ex: 워드프로세서가 디스크 종류를 신경쓸 필요 없이 쓰기를 진행할 수 있도록
  - 여러 프로그램이 동시에 도는 것처럼 보이게 하드웨어를 공유
  - 프로그램끼리 데이터를 공유하거나 협업할 수 있는 통제된(controlled) 방법을 제공
- 좋은 인터페이스 설계는 어렵다 -> 두 가지 사항을 다 만족시켜야함:
  - 인터페이스를 **simple & narrow** 하게 -> 구현을 제대로 하기 쉬움
  - 앱에 화려한 기능을 많이 주고 싶음
  - **_해법: 적은 수의 메커니즘을 활용하되, 그것들을 조합(combine)해서 큰 일반성(generality) 를 만들어내는 인터페이스를 설계하는 것_**
- 이 책의 **xv6** 라는 하나의 OS
  - Ken Thompson & Dennis Ritchie 의 Unix 가 제공한 기본 인터페이스를 따름 + 내부 설계도 흉내
  - 대부분(거의 전부)의 상용 OS들이 위 설계를 따르기에 xv6를 이해하면 다른 시스템들 이해의 좋은 출발점이 됨

- **Kernel** 이란?
  - 실행중인 프로그램들에게 서비스를 제공하는 특별한 프로그램 (Figure 1.1)
  - 컴퓨터에는 보통 많은 process가 있지만 kernel은 하나뿐
- **Process** 란?
  - 실행중인 하나의 프로그램
  - instructions(연산), data(연산이 다루는 변수), stack(프로시저 콜 구조) 을 담은 메모리를 가짐
- **System call** 이란?
  - process가 kernel 서비스를 호출해야할 때 부르는, OS 인터페이스에 정의된 call
  - system call이 kernel로 진입 -> kernel이 서비스 수행 -> 리턴
  - 따라서 process는 **user space** 와 **kernel space** 사이를 오가며 실행됨
- kernel은 CPU의 하드웨어 보호(protection) 메커니즘을 사용해, user space의 각 process가 자기 메모리만 접근하도록 강제
  - kernel은 이 보호를 구현하기 위해 하드웨어 특권(privilege)을 가지고 실행
  - user 프로그램은 그 특권 없이 실행
  - user 프로그램이 system call을 부르면 -> 하드웨어가 특권 레벨을 올리고, kernel에 미리 준비된 함수 실행
- kernel이 제공하는 system call의 집합 = user 프로그램이 보는 인터페이스 (Figure 1.2에 정의되어 있음)
  - xv6는 전통적인 Unix kernel이 제공하는 서비스/system call의 일부만 제공

- **Shell**:
  - 유저로부터 명령을 읽어 실행시키는 **user 프로그램 (kernel의 일부가 아님!)**
    - 이 사실이 system call 인터페이스의 강력함을 보여줌 -> shell은 특별할 게 없고 교체도 쉬움
      - 그래서 현대 Unix엔 다양한 shell이 존재
  - xv6 shell은 Unix Bourne shell의 핵심만 단순 구현한 것
- 이 챕터의 나머지는 process/memory/file descriptor/pipe/file system 에 대해 이야기하고, shell이 그것들을 어떻게 쓰는지 코드로 보여줌

## 1.1 Processes and memory
- xv6 process 구성:
  - user-space memory (instructions, data, stack)
  - kernel에만 있는 per-process state (private)
- xv6는 process들을 **time-share** 함
  - 실행 대기중인 process들 사이에서 가용 CPU를 투명하게 전환
  - 실행중이 아니면 process의 CPU register들을 저장해뒀다가 다음 실행때 복원

- **fork**: `int fork()`
  - 새 process를 만드는 system call
  - 부르는 process(부모)의 메모리를 **그대로 복사**한 새 process(자식)를 만듦 (instructions, data, stack 전부)
  - 부모와 자식 **양쪽에서 리턴**됨:
    - 부모에서는 자식의 PID를 리턴
    - 자식에서는 0을 리턴
  - 이 리턴값 차이로 "내가 부모냐 자식이냐"를 구분 (11pg 하단 참조)
  - 주의: 자식은 부모 메모리의 **복사본**으로 시작 -> 메모리/register가 분리됨
    - 한쪽에서 변수를 바꿔도 다른쪽엔 영향 없음
      - ex: 11pg 예시중 부모에서 `wait`의 리턴값이 `pid`에 저장돼도 자식의 `pid`(=0)는 안변함

- **exit**: `int exit(int status)`
  - 부르는 process를 멈추고 메모리/열린 파일 등 자원을 해제
  - `status` 인자는 
    - 관례적으로 0=성공
    - 1=실패.
  - 리턴 없음 (No return)

- **wait**: `int wait(int *status)`
  - 자식이 끝나길 기다리고, 끝난(혹은 killed된) 자식의 PID를 리턴
  - 자식의 exit status를 `status`가 가리키는 주소로 복사
    - exit status가 필요없으면 `wait`에 0(널 주소)을 넘기면 됨
  - 자식이 아직 안끝났으면 끝날때까지 대기
  - 자식이 아예 없으면 즉시 -1 리턴
  - ex: 11pg 예제의 출력 `parent: child=1234` 와 `child: exiting` 은 순서가 뒤바뀌거나 섞일 수 있음 (누가 먼저 printf에 닿느냐에 따라)
    - 자식이 exit한 뒤에야 부모의 `wait`가 리턴되어 `parent: child 1234 is done` 출력

- **exec**: `int exec(char *file, char *argv[])`
  - 부르는 process의 메모리를, 파일시스템에 저장된 파일에서 로드한 **새 메모리 이미지로 교체**
  - 파일은 특정 포맷이어야 함 -> 어디가 instruction이고 data이고 어디서 시작하는지를 명시
    - xv6는 **ELF 포맷** 사용 (자세한 내용은 [3장](chapter03.md)에서)
  - 성공하면 리턴하지 않음 -> 새로 로드된 instruction이 ELF 헤더의 entry point부터 실행됨
  - 인자: 실행파일 이름 + 문자열 인자 배열
  - 대부분의 프로그램은 `argv[0]`(=프로그램 이름)은 무시함

- shell이 위 call들을 조합해 프로그램을 실행:
  - main loop: `getcmd`로 한 줄 읽음 -> `fork` -> 부모는 `wait`, 자식은 명령 실행
  - 자식은 `runcmd` -> `exec`로 실제 명령(ex: echo)을 로드 -> 끝나면 `exit` -> 부모의 `wait`가 리턴

- 왜 `fork`와 `exec`가 분리되어 있나? -> "fork 직후 exec 직전" 의 **틈** 때문
  - shell이 이 틈에서 자식의 I/O를 redirection 할 수 있음 ([5장](chapter05.md)에서 자세히)
  - 단점: fork로 부모 메모리를 통째로 복사해놓고 바로 exec로 갈아엎으니 낭비스러움
    - 그래서 kernel은 **copy-on-write** 같은 가상메모리 기법으로 fork를 최적화 한다고함 ([5장](chapter05.md) 참고)

- **sbrk**: `char *sbrk(int n)`
  - process의 data memory를 0으로 채워진 `n` 바이트만큼 늘리고, 새 메모리의 시작 위치를 리턴
  - xv6는 대부분의 user 메모리를 암묵적으로 할당함:
    - fork -> 자식 복사본에 필요한 메모리 할당
    - exec -> 실행파일을 담을 만큼 할당
    - 런타임에 더 필요하면 (ex: malloc) `sbrk(n)` 호출

## 1.2 I/O and File descriptors

- **File descriptor(fd)** 란?
  - process가 읽거나 쓸 수 있는, kernel이 관리하는 객체를 나타내는 작은 정수
  - 파일/디렉토리/디바이스를 open 하거나, pipe를 만들거나, 기존 fd를 복제해서 얻음
  - fd가 가리키는 대상을 흔히 그냥 "file" 이라 부름
  - **file descriptor 인터페이스가 파일/파이프/디바이스의 차이를 추상화해서 전부 byte stream 처럼 보이게 함**
- 내부적으로 kernel은 fd를 per-process 테이블의 인덱스로 사용
  - 모든 process는 0부터 시작하는 자신만의 fd 공간을 가짐
  - 관례:
    - fd 0 = standard input (표준 입력)
    - fd 1 = standard output (표준 출력)
    - fd 2 = standard error (표준 에러)
  - shell은 이 관례를 이용해 I/O redirection 과 pipeline을 구현
    - shell은 항상 이 3개 fd가 열려있게 보장 ([소스코드](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/sh.c#L152))

- **read / write**:
  - `int read(int fd, char *buf, int n)`: fd에서 최대 n바이트를 buf로 읽고, 읽은 바이트 수를 리턴
    - 각 fd엔 offset이 연결돼 있음 -> 현재 offset부터 읽고, 읽은 만큼 offset 전진
    - 다음 read는 그 다음 바이트부터 -> 더 읽을게 없으면 0 리턴 (end of file)
  - `int write(int fd, char *buf, int n)`: buf에서 n 바이트를 fd로 쓰고, 쓴 바이트 수 리턴
    - read와 마찬가지로 현재 offset부터 쓰고 offset 전진 -> 각 write는 이전 write가 끝난 지점부터 이어씀
    - n만큼 write 하므로, n보다 적게 쓰이는 건 에러일 때만
  - ex: `cat`의 구현 ([표준입력(read) -> 표준출력 복사(write)](https://github.com/mit-pdos/xv6-riscv/blob/74f84181a3404d1d6a6ff98d342233979066ebb8/user/cat.c#L13))
    - `cat`은 자기가 파일에서 읽는지 console에서 읽는지 pipe에서 읽는지 모름 -> fd 추상화 덕분에 단순해짐

- **close**: `int close(int fd)`
  - fd를 해제해서 재사용 가능하게 만듦
  - 새로 할당되는 fd는 항상 그 process의 **가장 낮은 번호의 미사용 fd**
    - 이 규칙이 redirection 구현의 핵심이 됨

- **fork와 fd의 상호작용 -> I/O redirection 이 쉬워짐**:
  - fork는 부모의 fd 테이블도 함께 복사 -> 자식은 부모와 똑같은 열린 파일들을 가지고 시작
  - exec는 메모리는 갈아엎지만 fd 테이블은 보존
  - shell 에서의 구형: fork -> 자식에서 특정 fd를 close 하고 다시 open -> exec 의 순서로 redirection 구현
  - ex: `cat < input.txt` 를 shell이 실행하는 (단순화된) 코드:
    ```c
    char *argv[2];
    argv[0] = "cat";
    argv[1] = 0;
    if(fork() == 0) {
      close(0);                       // 표준입력 닫기
      open("input.txt", O_RDONLY);    // 가장 낮은 fd(=0)에 input.txt가 열림
      exec("cat", argv);
    }
    ```
    - 자식이 fd 0을 close 했으니 open은 그 자리(0)를 씀 -> `cat`은 표준입력이 input.txt를 가리킨 채로 실행
    - 부모의 fd는 안 건드림 (자식 fd만 수정했으므로)
    - [xv6 의 I/O redirection 의 실구현](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/sh.c#L83)
  - 왜 fork/exec가 분리돼야 redirection이 자연스러운가:
    - fork와 exec 사이에서 shell이 **메인 shell의 I/O는 그대로 둔 채** 자식의 I/O만 손볼 수 있음
    - 가상의 합쳐진 `forkexec` 였다면 redirection 옵션을 인자로 넘기거나 하는 등 어색해짐

- **open**: `open`의 두번째 인자는 비트 플래그 (fcntl.h 에 정의)
  - `O_RDONLY`(읽기), `O_WRONLY`(쓰기), `O_RDWR`(읽기+쓰기), `O_CREATE`(없으면 생성), `O_TRUNC`(길이 0으로 비움)

- **fd는 복사돼도 offset은 공유됨**:
  - fork가 fd 테이블을 복사해도, 그 밑의 **file offset은 부모-자식이 공유**
  - ex:
    ```c
    if(fork() == 0) {
      write(1, "hello ", 6);
      exit(0);
    } else {
      wait(0);
      write(1, "world\n", 6);
    }
    ```
    - 결과: fd 1에 `hello world` -> 부모의 write가 (wait 덕분에) 자식 write가 끝난 지점부터 이어씀
    - 이 offset 공유가 shell 명령 시퀀스의 순차 출력 (ex: `(echo hello; echo world) >output.txt`) 을 가능케 함

- **dup**: `int dup(int fd)`
  - 기존 fd를 복제해, 같은 I/O 객체를 가리키는 새 fd를 리턴
  - 두 fd는 **offset을 공유** (fork로 복제한 fd처럼)
  - ex: `hello world` 를 파일에 쓰는 또 다른 방법:
    ```c
    fd = dup(1);
    write(1, "hello ", 6);
    write(fd, "world\n", 6);
    ```
  - 두 fd가 offset을 공유하는 경우 = 같은 원본 fd에서 fork/dup으로 파생된 경우
    - 같은 파일을 각각 open한 fd들은 offset을 공유하지 않음
  - dup으로 가능해지는 shell 기능: `ls existing-file non-existing-file > tmp1 2>&1`
    - `2>&1` = fd 2를 fd 1의 복제로 만들라는 뜻 -> 정상 출력과 에러 메시지 둘 다 tmp1로 감
    - (xv6 shell은 error fd redirection은 지원 안하지만 원리는 이렇게 구현 가능)

- **fd가 연결된 대상(파일/console 같은 디바이스/pipe)을 숨겨주기 때문**에 file descriptor는 강력한 추상화

## 1.3 Pipes

- **Pipe** 란?
  - kernel이 들고 있는 작은 버퍼를, **읽기용/쓰기용 fd 한 쌍**으로 process에 노출한 것
  - 한쪽 끝에 쓰면 다른쪽 끝에서 읽힘 -> process간 통신(communication) 수단
- 16pg 코드, `echo "hello world" | wc` 를 직접 구현한 코드에 동작 해설
  - `wc`의 표준입력을 pipe의 읽기 끝에 연결
    - 부모가 "hello world"를 pipe에 쓰고, 자식이 `exec`을 통해 `wc`로 변해 그걸 읽어 줄/단어/바이트 수를 계산

  - **준비 단계**:
    - `int p[2];` -> 정수 2칸짜리 배열. pipe의 양쪽 끝 fd가 담길 곳
      - `p[0]` = 읽는 끝, `p[1]` = 쓰는 끝 (외우는 팁: 0=read, 1=write -> 표준입력/출력 순서와 동일)
    - `char *argv[2];` -> `wc`에 넘길 인자 목록 (문자열 2칸 배열)
    - `argv[0] = "wc";` -> 인자 0번은 항상 실행할 프로그램 이름
    - `argv[1] = 0;` -> 0(널)은 **"목록 끝"** 표시. C엔 배열 길이 정보가 없어 0을 만나면 끝인 줄 앎. `exec`가 이 0을 보고 멈춤

  - `pipe(p);` -> pipe 하나 생성. kernel이 내부 버퍼를 만들고 양쪽 (read/write) fd를 `p`에 채워줌
    - 실행 후 fd 상황: fd 0,1,2 = console / `p[0]`=fd 3(읽는 끝) / `p[1]`=fd 4(쓰는 끝)
    - `p[1]`에 write하면 그 데이터가 `p[0]`에서 read로 나옴 (단방향 통로)

  - `if(fork() == 0) { ... } else { ... }` -> fork로 자식 생성, fork() 리턴값으로 분기
    - 자식은 0z -> `if` 블록 / 부모는 자식 PID(양수) 리턴 -> `else` 블록
      - fork는 fd 테이블도 복사 -> 자식·부모 **둘 다** `p[0]`,`p[1]`을 손에 쥔 채 시작

  - **자식 블록 (if)**
    - wc가 되어 pipe에서 읽음
      - `close(0);` -> 표준입력(원래 console) 닫음 -> fd 0번 자리가 비워짐 (다음 줄을 위한 사전작업)
      - `dup(p[0]);` -> `p[0]`(읽는 끝)을 **가장 낮은 빈 fd**에 복제 -> 윗줄에서 close한 fd 0에 복제됨
        - 결과: fd 0이 pipe의 읽는 끝을 가리킴 -> `wc`가 표준입력을 읽으면 pipe를 읽게 됨
      - `close(p[0]);` 
        - 원래 `p[0]`(fd 3)은 중복이라 close
      -  `close(p[1]);`
        - `p[1]`(write end of pipe)은 자식에게 불필요하니 닫음
          - 안 닫으면 `wc`가 EOF를 감지못해서 멈춤
      - `exec("/bin/wc", argv);` -> 이 프로세스를 `wc`로 변경
        - exec는 **fd 테이블은 보존**한체 실행 -> fd 0 = pipe 읽는 끝이 그대로 살아있음

  - **부모 블록 (else)**
    - pipe에 hello world를 써서 자식의 wc에 전달
      - `close(p[0]);` -> 부모는 쓰기만 할 거라 읽는 끝은 닫음
      - `write(p[1], "hello world\n", 12);` -> 쓰는 끝에 데이터를 씀 
        - pipe를 타고 자식의 fd 0으로 흘러감
        - `12` = 바이트 수: `hello world`(11) + `\n`(1) = 12
      - `close(p[1]);`
        - 윗줄에 write에서 쓰기 작업이 종료되었으니 쓰는 끝도 닫음
        - 이제 pipe의 쓰는 끝을 가진 프로세스가 아무도 없음 -> 자식 쪽 `wc`의 read가 0(EOF)을 받아 정상 종료
  - 최종 결과: `wc`가 "hello world\n"을 받아 `1 2 12` (1줄, 2단어, 12바이트) 출력

- pipe의 read 블로킹 동작:
  - 데이터가 없으면 pipe의 `read`는 **데이터가 오거나, 쓰기 끝 fd가 전부 닫힐 때까지** 대기
  - 쓰기 끝이 전부 닫히면 `read`는 0 리턴 (= 데이터 파일의 EOF처럼)
  - 그래서 위에서 자식이 `wc` 실행 전에 **쓰기 끝(p[1])을 반드시 닫아야** 함
    - 안 닫으면 `wc`가 가진 쓰기 끝 fd 때문에 EOF를 영영 못 봄

- [xv6 shell의 pipeline 구현](https://github.com/mit-pdos/xv6-riscv/blob/riscv//user/sh.c#L101): 
  - ex: `grep fork sh.c | wc -l` 같은 것
    - 자식이 pipe를 만들어 좌측 끝과 우측 끝을 연결
    - 좌측 명령용으로 fork+runcmd, 우측 명령용으로 fork+runcmd, 둘 다 끝나길 wait
    - 우측이 또 pipe를 포함하면 (`a | b | c`) 재귀적으로 또 fork -> shell은 **process 트리**를 만들게 됨
      - 트리의 잎(leaf)은 명령, 내부 노드는 자식들을 기다리는 process

- **Pipe vs 임시 파일**: 
  - `echo hello world | wc` 는 임시 파일로도 흉내 가능
    - `echo hello world >/tmp/xyz; wc </tmp/xyz`
  - pipe가 나은 점 3가지:
    - pipe는 자동으로 정리됨
      - 임시파일은 `/tmp/xyz` 를 직접 지워야 함
    - pipe는 임의로 긴 데이터 스트림을 흘려보낼 수 있음 
      - 파일 redirection은 디스크 공간이 충분해야 함
    - pipe는 파이프라인 단계들이 병렬 실행 가능 
      - 파일 방식은 앞 프로그램이 끝나야 뒤가 시작

## 1.4 File system
- xv6 file system 제공하는 것:
  - data file: 해석되지 않은 byte 배열
  - directory: data file과 다른 directory에 대한 named references
- directory들은 root(`/`)에서 시작하는 트리를 형성
- **chdir**: `int chdir(char *dir)` -> 현재 디렉토리 변경

- 파일/디렉토리 생성 system call들:
  - **mkdir**: 새 디렉토리 생성 
    - `mkdir("/dir");`
  - **open + O_CREATE**: 새 data file 생성
    - `fd = open("/dir/file", O_CREATE|O_WRONLY); close(fd);`
  - **mknod**: 새 **device file(디바이스 파일)** 생성
    - `mknod("/console", 1, 1);`
    - 디바이스를 가리키는 특수 파일을 만듦
      - major / minor 디바이스 번호 두 개를 인자로 받음 -> kernel 디바이스를 유일하게 식별
        - 나중에 그 디바이스 파일을 open하면, kernel이 `read`/`write`를 (파일시스템 대신) kernel device implementationㅌ으로 전달

- 파일 이름은 파일 자체가 아님:
  - 같은 실제 파일(=**inode**)이 여러 이름(**links**)을 가질 수 있음
  - 각 link = 디렉토리의 한 엔트리 = (이름 + inode 참조)
  - inode: 파일에 대한 **metadata** 
    - 타입(파일/디렉토리/디바이스), 길이, 디스크상 콘텐츠 위치, link 수 등의 정보

- **fstat**: `int fstat(int fd, struct stat *st)`
  - fd가 가리키는 inode의 정보를 가져와 `struct stat`를 만듦 ([stat.h 에 정의된 구조](https://github.com/mit-pdos/xv6-riscv/blob/riscv//kernel/stat.h))

- **link**: `int link(char *file1, char *file2)`
  - 기존 inode를 가리키는 또다른 이름을 만듦
    ```c
    open("a", O_CREATE|O_WRONLY);
    link("a", "b");
    ```
    - 같은 inode 를 가진 경우, `a`를 읽고쓰는 것 = `b`를 읽고쓰는 것 
  - 각 inode는 유일한 **inode number**로 식별됨 -> 위 코드 후 `fstat`로 보면 `fstarta`,`b`가 같은 ino, `nlink`=2

- **unlink**: `int unlink(char *file)`
  - 파일시스템에서 이름 하나를 제거
  - inode와 디스크 공간은 **link 수가 0이 되고, 그 파일을 가리키는 fd도 없을 때만** 실제로 해제됨
  - ex: 이름 없는 임시 inode 만드는 관례적(idiomatic) 방법:
    ```c
    fd = open("/tmp/xyz", O_CREATE|O_RDWR);
    unlink("/tmp/xyz");
    ```
    - unlink를 통해 이름은 지웠지만 fd가 살아있어서 이후 라인에서 0,1 번 fd로 접근 가능
    - process가 fd를 닫거나 종료하면 자동 정리되어 해당 데이터도 삭제됨

- 파일 유틸리티는 user-level 프로그램:
  - `mkdir`, `ln`, `rm` 등은 shell에서 부르는 평범한 user 프로그램
    - 누구나 새 user 프로그램을 추가해 command-line 인터페이스를 확장 가능
    - 당시 다른 시스템들은 이런 명령을 shell 안에 빌트인했음
      - shell 도 커널에 직접 빌트
  - **`cd` 는 shell에 빌트인됨**
    - `cd`는 shell 자신의 현재 디렉토리를 바꿔야 함
    - 만약 `cd`가 일반 프로그램이었다면? 
      - shell이 fork한 **자식**의 디렉토리만 바뀌고 부모(shell)는 그대로 -> 유저 입장에서 아무 의미 없는 행위가 되어버림

## Real world
- Unix의 조합 = "standard" file descriptor + pipe + 편리한 shell 문법 -> 범용 재사용 프로그램 작성의 큰 진보
  - "software tools" 문화를 촉발, shell이 최초의 "scripting language"
  - 이 system call 인터페이스는 BSD/Linux/macOS에 지금도 이어짐
- **POSIX** (Portable Operating System Interface) 표준으로 표준화됨
  - xv6는 POSIX 호환이 아님
    - 많은 system call이 빠져있고(ex: `lseek`), 제공하는 것도 표준과 다를 수 있음
  - xv6의 목표는 단순한 Unix-like system-call 인터페이스 구현
  - 현대 kernel은 networking/windowing/user-level thread/디바이스 드라이버 등 훨씬 많은 기능 + POSIX 이상으로 계속 진화
- Unix는 여러 종류의 자원(파일/디렉토리/디바이스)을 **하나의 파일-이름+fd 인터페이스로 통일** 접근
  - 이 아이디어를 더 밀어붙인 예: **Plan 9** -> "resources are files" 를 네트워크/그래픽까지 적용
    - 다만 대부분의 Unix 계열은 이 길을 따르진 않음
- **xv6엔 user 개념이 없음** -> 한 user를 다른 user로부터 보호하지 않음 -> 모든 xv6 process가 root로 실행됨
- 이 책은 xv6가 Unix-like 인터페이스를 어떻게 구현하는지를 보지만, 그 개념은 Unix 너머에도 적용됨:
  - 어떤 OS든 process를 하드웨어에 multiplex 하고, process를 서로 격리하고, 통제된 inter-process 통신을 제공해야 함

## Exercises

```c
#include "kernel/types.h"
#include "user/user.h"

int main() {
  int parentPipe[2]; // 부모 -> 자식 방향
  int childPipe[2];  // 자식 -> 부모 방향
  pipe(parentPipe);
  pipe(childPipe);

  char ping = '1';
  char pong = '2';
  char buf;

  if (fork() == 0) {
    close(parentPipe[1]); // 자식은 부모쪽 파이프에 쓰지 않음
    close(childPipe[0]);  // 자식은 자기쪽 파이프에서 읽지 않음

    for (int i = 0; i < 1000; i++) {
      if (read(parentPipe[0], &buf, 1) != 1)
        break;
      write(childPipe[1], &pong, 1);
    }

    close(parentPipe[0]);
    close(childPipe[1]);
    exit(0);
  } else {
    close(parentPipe[0]); // 부모는 부모쪽 파이프에서 읽지 않음
    close(childPipe[1]);  // 부모는 자식쪽 파이프에 쓰지 않음

    int start = uptime();
    for (int i = 0; i < 1000; i++) {
      write(parentPipe[1], &ping, 1);
      if (read(childPipe[0], &buf, 1) != 1)
        break;
    }
    int end = uptime();

    close(parentPipe[1]);
    close(childPipe[0]);
    wait(0);

    // 정수 연산: 1 tick = 100ms = 100000us
    long ticks = end - start;
    long total_us = ticks * 100L * 1000L;
    printf("%ld ticks (~%ld ms), 1 round trip takes about ~%ld us\n", ticks, ticks * 100, total_us / 100000);
  }

  exit(0);
}
```
- 설명:
  - 2개의 파이프를 생성하고 fork를 통해 자식 프로세스를 생성 후 부모와 자식간에 ping 과 pong을 전달하며 변경시키는 로직.
  - 1000번 루프하는 이유는 user.c 에 정의되어 있는 uptime이 리턴하는 값이 틱 단위의 정수만을 리턴하기에 하나의 rtt에 대한 정확한 값을 측정이 불가함.
  - 