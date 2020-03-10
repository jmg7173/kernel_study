# Interrupt & Interrupt Handler

## Introduction
모든 OS는 Host machine에 연결되어있는 Hardware 기기(ex. HDD, SSD, 마우스, 키보드 등)들을 관리할 수 있어야 한다. 이를 위해 Kernel은 Host system에 연결되어있는 장치들과 통신할 수 있어야한다. 보통 processor(i.e. CPU)들은 이 Hardware들보다 수십 배 빠르므로 Hardware의 응답을 기다리는건 매우 비효율적이다. 그래서 Hardware가 실제로 작업을 완료한 경우에만 응답을 처리하고, 그 사이에 다른 작업들을 수행하도록 하는 것이 효율적이다.

Hardware의 작업 상태를 다루는 방법에는 크게 두 가지가 있다.

* Polling
* Interrupt 

Polling은 kernel이 주기적으로 Hardware의 상태를 확인하는것으로, Hardware가 활성화되지 않았거나 준비되지 않았음에도 계속 반복적으로 상태를 물어보는 오버헤드가 존재한다.

Interrupt는 하드웨어가 kernel에 신호를 보내는 것이다. 따라서 interrupt는 polling과 달리 계속해서 요청을 보내지 않아도 되기 때문에, host system에 가해지는 부하가 매우 적다. 하지만 polling보다는 응답속도가 다소 느리다는 단점이 있다.

이 챕터에서는 interrupt가 무엇인지, kernel이 interrupt handler를 통해 어떻게 interrupt를 다루는지에 대해 살펴본다.

## Interrupt
Interrupt의 종류에는 Software Interrupt와 Hardware Interrupt가 있다.

### Hardware Interrupt
Hardware interrupt는 Hardware가 processor에 전기적인 신호를 보내는 것이다. 예를 들어, 키보드의 키를 누르면 processor로 전기적인 신호를 전달하여 OS가 누른 키를 인식하도록 할 수 있는데, 이것이 바로 hardware interrupt이다. Hardware interrupt는 언제든지 발생할 수 있고, 비동기적으로 생성한다. 이를 처리하기 위해 kernel은 언제든지 interrupt될 수 있다.

Hardware에서 생성한 전기적인 신호를 통해 생성된 interrupt는 interrupt controller를 거쳐 processor로 전달된다.

<!-- 그림 삽입 -->
![Hardware Interrupt](http://jake.dothome.co.kr/wp-content/uploads/2017/01/interrupt-1a.png)

모든 Hardware는 interrupt line을 통해 interrupt controller와 interrupt line으로 연결되어있다. Interrupt controller는 multiplexer와 같은 역할을 하여 연결된 여러 interrupt line의 input을 processor와 연결된 하나의 line으로 전달한다. Interrupt controller로부터 interrupt가 전달되면 processor는 이를 감지해 현재 실행중인 작업을 잠시 중단하고 interrupt를 처리한다. 그리고 processor는 OS에 이를 알려 처리하도록 한다.

Device 종류마다 고유한 interrupt 값을 가지며, OS는 이를 통해 interrupt를 구별해 이에 연결된 interrupt handler를 통해 interrupt를 처리한다. 이 값들은 interrupt requests(IRQs) line이라고 하며, 각 line에는 값이 할당되어있다. Timer는 0, Keyboard는 1 이런식으로 정적으로 할당 되어있는 경우도 있으나, PCI device의 경우에는 동적으로 값이 할당된다. 어찌됐든 Kernel은 각 interrupt value에 매칭되는 interrupt handler 및 device를 알고 있다.

### Software Interrupt
Hardware interrupt는 비동기적인 interrupt고, system calld이나 exception의 경우는 동기적인 interrupt이다. Exception의 예로는 divide by zero와 같은 programming 오류, page fault와 같이 커널에서 처리해야할 비정상적인 조건들이 있다. 대부분 Hardware interrupt와 비슷한 방식으로 처리한다. 다만, system call을 호출하는 `int 0x80`과 같이 assembly code를 통해 기계어 명령을 통해 실행된다.

## Interrupt Handler
Kernel에서 Hardware등에 대한 interrupt를 처리하는 것을 interrupt handler 또는 ISR(Interrupt Service Routine)라고 한다. Interrupt를 생성하는 장치는 각각 연결된 interrupt handler가 있으며, 보통 Device Driver에 포함되어있다. Device driver는 다른 장에서 다루며, 간략하게 말하면 device를 다루는 kernel code정도로 이해하면 된다.

Interrupt handler를 통해 interrupt를 처리하는 것은 interrupt context라 하는 특수한 context에서 수행된다. 이 context에서 code가 수행될때는 block(ex. sleep, schedule 등 스스로 block되는 행위)할 수 없다. 하지만 interrupt가 수행되는 동안 다른 interrupt에 의해 중단될 수도 있다. 이는 interrupt handler 옵션에 따라 다르다.

같은 종류의 다른 device에서 동일한 interrupt handler를 통해 interrupt를 처리하는 경우 interrupt line과 interrupt value가 같다. Interrupt handler가 실행중일 때 같은 interrupt에 대한 처리가 불가하다. 즉, 해당 interrupt line이 masking되어 비활성화가 된다. 다른 interrupt line은 비활성화 되지 않으므로 interrupt handling이 가능하다. 따라서 nested interrupt가 발생하지 않으며, 동일한 interrupt handler를 동시에 호출하지 않는다.

Interrupt는 언제든지 발생할 수 있다. Interrupt가 발생하면 수행중이던 프로그램은 중지되므로 가능한 빨리 재개하도록 interrupt handler가 신속하게 interrupt를 처리하는것이 중요하다. 그러나 interrupt handler에서 일을 많이 처리해야해서 빠르게 처리 못하는 작업이 있을 수도 있다. 예를 들어 network stack을 거쳐 packet을 처리해야하는 작업이 있다. Interrupt handler에서 꼭 수행해야하지만 시간이 오래 걸리는 것을 interrupt context에서 처리하면 다른 중요한 작업들을 수행할 수 없기 때문에, interrupt의 작업을 다음과 같은 두 가지 부분으로 나눈다.

* Top Halves

Interrupt가 일어나면 즉시 실행되는 부분으로, interrupt 수신 확인이나 Hardware 재설정과 같은 중요한 작업만 우선적으로 수행한다.

* Bottom Halves

나중에 수행해도 되는 작업들로, Top halves를 수행하고 남은 Bottom halves는 다양한 방법으로 OS scheduler에 의해 scheduling되어 나중에 처리된다.

## Interrupt Context
OS kernel에서 context는 두 가지로 나뉜다.
* Process Context
* Interrupt Context

프로그램 코드를 수행하면 process가 생성이된다. Executable file이 실행되면 program code를 읽어서 program의 address space내에서 실행한다. 일반적으로 program의 실행은 "**user space**"에서 수행된다. Program 실행 중 System call 호출 등을 통해 kernel에 진입하면 "**kernel space**"로 전환된다. 이 시점에서 kernel은 'process 대신 수행되고 있는 "**process context**"에 있다'고 한다. Process context에 있는 kernel thread는 sleep이 가능하고 선점가능하다. 즉, `schedule()`을 통해 scheduler를 불러 schedule out될 수 있다.

Interrupt context는 interrupt handler를 실행하는 경우의 kernel thread의 context이다. 이 상태에서는 sleep이 불가하는 등 process context인 상태에서 호출할 수 있는 kernel 함수의 일부를 사용할 수 없다. Interrupt handler는 자신의 stack을 가질 수 없고, interupt된 process의 stack을 공유한다. 보통 kernel stack은 두 페이지(8KB or 16KB)로 제한되어있으므로, 이 context에서는 데이터 할당을 더 유의해야한다.
