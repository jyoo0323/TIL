# Chapter 1 Introduction
- **Operating System** is a software that messages a computer's hardware.
- 4 components of computer system:
  1. Hardware: CPU, memory, input/output devices, etc.
  2. Application Programs: tools that solve users' computing problems.
  3. Operating System: controls hardware and coordinates hardware among various programs for various users
  4. User: 사용자 그 자체

### 1.1.3 Defining Operating Systems
- 딱 정의할 수 있는 명확한 정의는 없음
- 컴퓨터의 역사를 알아야하는데, 컴퓨터 시스템이 발전을 해온 목적은 사용자의 문제를 해결하는 것을 돕기 위함이고, 이 과정에서 프로그램이 생겨났으며, 이러한 프로그램들이 공통적으로 항상 하는 작업 (ex:I/O 장비의 컨트롤) 등을 모아 리소스를 컨트롤하고 할당하는 프로그램 => Operating System. 

### 1.2.1 Interrupts
- I/O 장치를 컨트롤하는 디바이스 컨트롤러가 자신에게 작업을 요청한 디바이스 드라이버에게 자신의 작업이 완료(write completed successfully)되었거나 device busy 등의 정보를 알리는 작업/시그널.

Figure 1.3 (8 pg) 의 해석:

1. **I/O Request (입출력 요청)** CPU(사용자 프로그램)가 I/O 장치에 데이터를 출력하도록 요청. 이 시점에 CPU는 I/O 컨트롤러의 레지스터를 설정. 이후 장치는 데이터를 전송하기 시작하고, CPU는 바로 사용자 프로그램의 다음 작업을 계속 수행.
2. **I/O Device: transferring** I/O 장치는 이제 데이터 전송을 수행 중. CPU는 이 전송이 완료되길 기다리지 않고 다른 계산을 계속합니다 (비동기적 수행).
3. **Transfer done** I/O 장치가 전송을 완료. “작업이 끝났음”을 알릴 준비가 된 상태.
4. **Interrupt signaled** I/O 장치 컨트롤러가 CPU에 **인터럽트 신호**를 전송. CPU는 현재 실행 중이던 사용자 프로그램을 잠시 중단하고 인터럽트 처리 루틴을 수행. 
5. **Interrupt handled** CPU는 interrupt vector 에 있는 기존 주소(fixed location)로 점프하여 해당 장치의 인터럽트를 처리. 이 스텝에서는 장치 상태를 확인하고, “write completed successfully” 상태를 갱신
6. **Return to user program**  인터럽트 처리가 끝나면 CPU는 원래 실행 중이던 프로그램의 상태를 복구하고, 이어서 사용자 프로그램을 계속 실행합니다.
7. **다음 I/O 요청 반복** 프로그램이 다시 새로운 I/O 작업을 요청하면, 위의 순서가 다시 반복

- 즉 이미 CPU 는 비동기-nonblocking 하게 다른 작업을 프로세싱하고 있고, I/O 장비가 데이터를 전송하는 동안은 다른 작업을 수행하다가, interrupt 가 signaled 될 경우에 다시 기존 작업으로 돌아와 I/O 장비의 완료된 작업을 마무리하는 방식. (어떻게 마무리하는 가는 나중에 볼 수 있을 듯 하다.)
