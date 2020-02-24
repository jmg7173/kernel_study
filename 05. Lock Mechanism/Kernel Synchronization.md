
# PRINCIPLES
최근 대부분의 시스템의 특성인 **Preemptive Kernel** / **Shared-multiprocessor, or multicores**로 인해  커널코드는 동시에 수행될 수 있다 **(Concurrency, 동시성)**.   

이러한 멀티프로세서 프로그래밍은 동시성 문제를 해결하기 위한 수 많은 challenge를 안고있다.
> **Shared resources require protection from concurrent access** because if multiple threads of execution access and manipulate the data at the same time, the threads may overwrite each other’s changes or access data while it is in an **inconsistent state**. Concurrent access of shared data is a recipe for instability that often proves hard to track down and debug—getting it right at the start is important.
## Keywords
   
- **Critical section**: 공유되는 데이터를 접근하여 조작하는 코드 부분, 두개 이상의 스레드에서 실행되지 않아야 한다.   

-  **Race condition**: 여러 프로세스가 Critical section에 들어간 혹은 들어갈 수 있는 상태, 데이터의 일관성을 해칠수 있다.
	>  Race condition scenario (**Appendix I**)
- **Mutual Exclusion (상호배제)**: Critical section에 여러 스레드가 동시에 들어가는 것을 방지하려는 속성
  >The standard way to approach mutual exclusion is through a "Lock object".
- **Synchronization (동기화)**:  Mutual exclusion 알고리즘과 같은 방식을 이용해 Race condition을 예방하는 것
- **Atomic (원자성)**: 더 이상 쪼개질 수 없는 성질. 어떠한 작업이 실행될 때, 언제나 완전히 진행되어 종료되거나 그럴 수 없는 경우 실행을 하지 않는다. (All or Nothing)
	- 원자성이 없는 명령어들은 **Consistency** (**일관성**)을 해칠 수 있다. 
	- 예를 들어:  **0으로 초기화 된 int형 공유 변수 i가 있을때, "i++;" 를 두 스레드가 동시에 수행한다고 가정**
	- 프로세서에서 이 명령어를 수행하기 위해 다음과 같은 동작을 수행 할 것이다.
	> 1. 변수 i의 값을 레지스터에 복사
	> 2. 레지스터에 저장된 값에 1을 더함
	> 3. 메모리에 새 i 값을 기록   	
	- **즉,  i++을 수행하는 instruction은 atomic하지 않다.**      
	
		 ![Shared data가 inconsistent한 예시, Critical section](https://lh3.googleusercontent.com/ChyPvs2v9S-W62z2oRlIIMjUS5UwWMHOjuP15hcybU2fau2WIiYgBjCXRGgn-wYaPjCTHnAiXJIv)
	
	> 원자적이지 않은 동작으로 인해 기대한 결과 값 2가 아닌 1이 결과 값이 될 수 있다. 
	이러한 상황을 예방하기 위해 커널에서 제공하는 **atomic operation**을 사용하거나 **lock**을 사용해야한다.
-  **Locking**:  Critical section이 atomic하게 수행될 수 없는 상황이라면 (예를들어 단순한 산술, 비교 연산이 아닌 큐 갱신 같은 동작) critical section 시작과 끝을 표시하여 이 구역이 실행되는 동안 다른 스레드의 접근을 막을 수 있는 방법, 즉 **Locking**이 필요하다.    
이후에 다양한 lock의 특성과 종류를 살펴본다.
-  **Deadlock**: 각 스레드들은 자원을 기다리지만 (wait) 모든 자원들은 이미 보유된 상태 (held).     
즉 스레드들은 critical section에 진입하기를 기다리지만, 모든 critical section들은 lock 되어 있다.
모든 스레드가 서로에 대해 대기하고 있어 그들이 잡고 있는 자원을 놓지 않는다.   
	- **Deadlock 간단한 예시 (ABBA deadlock)**
			![](https://lh3.googleusercontent.com/2N5C4_IhzxdB13No8A9c8AXp89nadTiQ4n2L1k_X8G4jWfOR9ha2B8ZFL5agL33Z-AroieNhfyBz "ABBA deadlock")

- **Lock contention**: 이미 한 스레드가 acquire한 lock을 다른 스레드가 acquire하려고 하는 상황 (wait). Wait하는 스레드들이 많을 수록 contention은 심해진다.   
lock은 어떤 자원으로의 접근을 serialize하기 때문에 성능 저하를 시키는 것은 당연하지만, 경쟁이 심한 lock일 수록 더 큰 저하를 발생시킨다.
  >특히 Lock의 성능은 **scalability (확장성)** 와 연관이 깊다.    
-  **Lock granularity (세세함)**:  Lock이 보호하는 데이터, 코드의 크기.    
granularity가 작을수록, 즉 locking이 세분화 될 수록 **성능 확장성**이 증가한다.
	- granularity가 큰 lock을 **coarse grained lock**, 작은 lock을 **fine grained lock**이라고 한다.
	- 예를 들어 스케줄러의 실행큐 (run-queue)는 Linux kernel 2.4버전에서는 하나의 큐 (single queue)였지만 2.6버전에선 CPU마다 큐 (per-cpu queue)를 갖게 되었다. 큐를 보호하는 락의 granularity가 매우 줄어 성능은 선형적으로 증가하였다.
	> 대체로 Coarse grained lock을 Fine grained lock으로 변형하는 것은 확장성에 긍정적인 영향을 미치지만, lock contention이 적은 경우 오히려 세분화된 lock이 오버헤드로 작용할 수 있다.

## Synchronization Methods

- **Synchronization approaches in the Kernel**

	- **For single core**
		-  Disable preemption or interrupts (**Appendix II**)
	- **For multi-core**
		-  Atomic operations to perform multiple actions at once
		- Locking primitives to prevent other tasks from entering a critical section   

### Atomic operation
 - Atomic operation들은 
	- 중단됨 없이(without interruption) atomically하게 수행되는 instruction들을 뜻한다. 
		-  **Atomic operation example**
		![enter image description here](https://lh3.googleusercontent.com/0ijrODnDr2o5GeyX6VEatmGOWzx0aKd7nnZZvoDPCpCbDP480qdVcZsIh2NEwd-KgYjjKdndYkWU)
	- 가장 작은 단위의 동기화 기법으로 가장 빠른 성능을 보인다.
	- 전용 자료구조인 atomic_t를 사용한다.
```   
typedef struct {    
    int counter;   
} atomic_t;
```
- Linux kernel system에서 제공하는 atomic operation들 예시:
	-  ***atomic_t***  v  = **ATOMIC_INIT(0)**;
	- ***atomic_set(&v, 4);*** /* v = 4 (atomically) */
	- ***atomic_add(3, &v);*** /* v = v + 3 (atomically) */
	- ***atomic_inc(&v);*** /* v = v + 1 (atomically) */
	> 아키텍처들은 커널에서 사용가능한 최소한의 atomic 함수를 지원해야 한다.
	 **Appendix III**: Code level의 부연 설명
	 - 간단한 atomic 연산만으로 처리할 수 없는 상황에선 보다 복잡한 동기화 방법, 즉 Lock이 필요하다.
	 > **Appendix IV**: 대부분의 락 구현은 내부적으로 atomic operation들을 활용한다.
### Spin Locks

- 프로세스가 락이 사용 가능해질 때까지 **busy-waiting**하는 **mutual exclusion mechanism**
>**What do you do if you cannot acquire the lock?**   There are two alternatives.
>1. If you keep trying, the lock is called a **spin lock**, and repeatedly testing the lock is called **spinning, or busy waiting.** (sensible when you expect the lock delay to be short)
>2. Suspend itself and schedule-out, which is called **blocking.**
- 아키텍처에 종속적이며(architecture-dependent) 어셈블리로 구현되어 있다. 
- Spinlock은 커널에서 가장 일반적으로 사용되며, **인터럽트 핸들러에서도 사용 가능하다**. (Blocking, sleep 되는 락들은 사용이 불가하다.)
	-  인터럽트 핸들러에서 사용할 경우, lock acquire 전 반드시 **local interrupt를 비활성화** 해야한다. ( ***local_irq_save()*** )
		-  	만약 그렇지 않으면 인터럽트가 이미 lock을 잡고 있는 코드를 중단 후 같은 lock을 잡으려는 다른 코드를 수행해 dead-lock 상태에 빠질 수 있다. 
		(*참고: Preemption은 꺼진 상태)
-	Spin lock implementation (Multicore, CONFIG_SMP is set)
	-	**Spinlock** 
		- __raw_spin_lock(*lock):
		***preempt_disable()*** 수행 후 ***do_raw_spin_lock(...)*** 수행
		- __raw_spin_unlock(*lock): 
		***do_raw_spin_unlock(...)***  수행 후 ***preempt_enable()***  수행
	-	**Spinlock in an ISR**
		- __raw_spin_lock_irqsave(*lock, flag): 
		***local_irq_save()*** 수행 후 ***preempt_disable()*** , ***do_raw_spin_lock(...)*** 수행
		- __raw_spin_unlock_irqrestore(*lock, flag):
		 ***do_raw_spin_unlock(...)*** 수행 후 ***local_irq_restore()*** , ***preempt_enable()*** 수행
- ***do_raw_spin_lock*** 에 대한 커널 버전별로 성능 최적화를 위한 다양한 구현이 존재 (다음 장에서 다룸)

### Semaphores / Mutexs
- **Semaphores / Mutexs** are sleeping lock
	- 어떤 task가 이미 잠겨진 세마포어를 잠그려 하는 경우 세마포어는 그 task를 대기 큐에 삽입하고 task를 sleeping 상태로 만든다. 프로세서는 다른 코드를 실행 할 수 있게 된다.
- Lock을 짧게 잡는 상황에는 spinlock을, 시간이 길 것으로 예상되는 경우 semaphore를 사용하는 것이 유리하다. (Context-switching overhead 고려)
- 인터럽트 컨택스트는 스케줄링이 불가하므로 semaphore를 사용할 수 없다. 
Lock  contention 시 sleep 해야하기 때문이다. 달리 말하면 세마포어는 커널 선점을 비활성화 할 필요가 없다.
- Semaphore는 counter를 가지고 있어 최대 semaphore holder 수를 관리할 수 있다.
- 이 counter 값을 1로 설정하는 경우 binary semaphore 혹은 mutual exclusion을 강제한다는 점에서 **Mutex**라고 불린다.
	- Semaphore에서는 counter 값이 1보다 큰 경우가 존재할 수 있는데 이 경우 다수의 스레드가 동시에 critical section에 진입할 수 있으므로 mutual exclusion을 보장하지 않는다.
- Semaphore/Mutex implementation
	- Data structure
	```
	struct semaphore{
	    raw_spinlock_t lock;
	    unsigned int count;			// the number of permissible simultaneous holders of semaphores
	    struct list_head wait_list; // wait queue for tasks which failed to acquire the semaphore
	}
	struct mutex{
	    raw_spinlock_t lock;
	    atomic_t count;			// 1: unlocked, 0: locked, <0: locked, possible waiters
	    struct list_head wait_list; // wait queue for tasks which failed to acquire the mutex
	}
	```
	- Semaphore implementations rely on spin locks (커널에서 많이 쓰이지 않는다)
	```
	Down_interruptible(...){
		spin_lock();
			if(sem->count > 0)
				sem->count--;	// acquire lock
			else
				__down_interruptible();	// Put the task into the semaphore wait list and sleep
		spin_unlock();
	}
	```   
	
	- Spinlock은 preemption을 disable 시키기 때문에 spinlock을 잡은 상태로 down()을 수행하면 task는 wakeup 할 수 없다. 따라서 down_interruptible() 함수 내부에서 ***spin_unlock-> schedule_out-> spin_lock*** 을 수행한다. (즉, 매우 비효율적)
- Mutex implementations은 다음 장에서 다룸.

### Readers-Writers Problem

- 데이터가 update (쓰기) 되는 경우에는 mutual exclusion이 필요하지만 search (읽기)의 경우 필요로 하지 않는다.
- Lock의 사용이 reader와 writer로 구분되는 경우 이러한 특성을 활용할 필요가 있다.
- Spinlock과 Semaphore 두 방식으로 모두 구현할 수 있다.
- Reader와 Writer 중 누구에게 우선권을 줄 것인가? (**Appendix V**)
	- First readers-writers problem
		- Reader에게 우선권을 주는 방식
		- Writer는 unbounded waiting을 겪을 수 있다. (starvation)
		> - No reader will be kept waiting until a writer has already obtained permission to user the shared object.
		> - No reader should wait for other readers.
	- Second readers-writers problem
		- Writer에게 우선권을 주는 방식
		- Reader는 unbounded waiting을 겪을 수 있다. (starvation)
		> - Once a writer is ready, that writer performs its write as soon as possible.
		>- If a writer is waiting to access the object, no new readers may start reading.
	
- **Sequential Locks**
	- Second RW problem과 마찬가지로 writer에게 우선권을 주는 방식
	- Sequence counter를 사용하여 구현 ( **Appendix VI** )
		- Writer는 Lock과 Unlock 시 counter를 증가시킨다. (초기 값 = 0)
		- Reader는 Lock과 Unlock 시 counter를 읽고 비교한다.
		- 만약 전후로 읽은 counter 값이 다르다면 read 수행 중 write가 수행 되었다는 의미이므로 retry 
		- counter 값이 짝수인지 홀수인지에 따라 writer가 수행 중인지 아닌지 구분 가능하다.
	- 많은 Reader와 적은 Writer가 있는 경우 가볍고 확장성이 좋은 기법
	- Writer는 다른 writer가 없다면 항상 lock을 얻을 수 있다.
	
### RCU (Read-Copy-Update)
-	Reader side overhead를 최소화하는데 목적이 있어 읽기 동작이 많은 경우에 사용한다. (Update operation 비율이 10 % 이상일 경우 오히려 성능 하락)
-	Reader는 shared data에 lock 없이 접근할 수 있다. (rwlock은 writer가 없을 떄만 reader 수행을 허락하기 때문에 lock이 필요)
-	Writer는 data를 copy하고 update 후 pointer를 변경한다.
-	**RCU concept**
	1. Write 시에 original data를 복사한 후 copied-data에 write 한다.
	2. Reader가 아무도 없을 때, copied data -> original data로 replace한다.
	3. original-data는 더이상 쓸모가 없기 때문에 메모리에서 해제한다.
- RCU를 이루는 3가지 요소
	1. Reader: RCU-protected data를 read한다. Read 전후로 reference counter를 관리한다.
	2. Updater: Writer의 역할. RCU-protected data를 변경한다. Copy and Update. 
	Reader가 없을 때까지 대기 후, data replacement 역할도 수행. 
	Write operation 전후로 동기화 필요 (spinlock과 같은 락)
	3.  Reclaimer: RCU-protected original data를 폐기한다.
> Grace period: Old 혹은 original data에 대해서 더이상 reference하는 reader가 없을 때까지 기다리는 기간. 
> Grace period 가 끝나는 시점을 감지하기 위해 CPU는 
-	자세한 내용은 다음 장에서 다룸
# PRACTICE
	Preview
	1. TAS spinlock
	2. ticket spinlock
	3. Qspinlocks (MCS)	
	4. Mutex
	5. RCU 
