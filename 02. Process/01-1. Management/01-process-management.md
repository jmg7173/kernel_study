# Process Management
## 목차
- [Process 란](#the-process)
- [Process 세부 정보](#process-descriptor)
(PID, Process State, Process Family Tree)
- [Process 생성](#process-creation)
(Process 실행)
- [Thread 생성](#thread-creation)
(Thread 실행)
- [Process 종료](#process-termination)

## The Process
**Process**의 정의: 실행중인 코드, 실행중인 프로그램

Process의 의미: 실행중인 프로그램 + 관련된 자원들, 실행중인 코드의 결과물

Process는 생성되었을 때 life cycle이 시작된다.

Linux에서 **Thread**는 특별한 종류의 process이다. (Thread 관련 자세한 내용은 [Thread 생성](#thread-creation) 항목으로)
Process는 하나 이상의 thread로 이루어져 있으며, 커널은 process가 아닌 thread 단위로 스케줄링을 진행한다.

**참고)** **Task**는 작업의 최소 단위를 일컫는 용어이다. 즉 task는 process를 지칭할 수도, thread를 지칭할 수도 있다. 

## Process Descriptor
커널은 circular doubly linked list 구조인 `task list`에 process 세부 정보들을 저장한다. `Task list`의 각 원소들은 `struct task_struct` 타입의 구조체이며, 이들을 **Process Descriptor**라고 한다.
<br></br>
**<위치>**

각 process는 커널 영역에서 사용할 스택인 process kernel stack을 가진다. 이 스택의 최상단 주소에 `struct thread_info` 구조체가 위치된다.
- 장점: 별도의 레지스터를 사용하지 않고 현재 process의 process descriptor에 접근할 수 있다.

~~~
struct thread_info {
	struct task_struct *task;
	int preempt_count;
	struct restart_block;
	void *sysenter_return;
	...
}
~~~
이 구조체의 `task`는 `struct task_struct *` 타입으로, 해당 process의 실제 process descriptor를 가리킨다.
<br></br>
**<접근>**

현재 process의 process descriptor에 접근하는 방법으로, `current` 매크로가 구현되어 있다. 이 매크로는 어셈블리 코드로 구현된 `current_thread_info()` 함수를 통해 process kernel stack의 `thread_info` 구조체에 접근한 뒤, 포인터를 통해 `task_struct`를 가리키도록 한다.
<br></br>
**<구성>**

<p align="center">

<img src = "https://img1.daumcdn.net/thumb/R800x0/?scode=mtistory2&fname=https%3A%2F%2Ft1.daumcdn.net%2Fcfile%2Ftistory%2F9986143359D9CE5D14">

</p>

실제 구조체 코드: [<linux/sched.h>](https://github.com/torvalds/linux/blob/f8788d86ab28f61f7b46eb6be375f8a726783636/include/linux/sched.h#L629)

***PID***

각 Process를 구분하기 위해 `pid_t` 타입의 `pid` 변수를 가진다. `pid_t` 타입은 process 번호를 저장한다는 의미를 가지며, `pid`는 양수의 integer 값으로 표현된다. `PID_MAX_DEFAULT`는 32,768로 정해져있다. (`32bit short int`)

***Process State***

각 Process의 상태를 `unsigned long` 타입의 `state` 변수에 지정한다.
1. `TASK_RUNNING`: Ready queue에서 대기하거나, CPU를 점유하여 running중인 상태이다.
- User-space에서 실행중인 process는 모두 `TASK_RUNNING` 상태이다.
2. `TASK_INTERRUPTABLE`: Process가 블락된 상태로, 신호를 받으면 (조건에 관계없이) 다시 깨어나 `TASK_RUNNING` 상태가 될 수 있다.
3. `TASK_UNINTERRUPTIBLE`: Process가 블락된 상태로, 특정 조건을 만족하면 다시 깨어나 `TASK_RUNNING` 상태가 될 수 있다.
- Process가 휴면 상태에 진입할 경우, 기본적으로 2번 상태가 된다. 이때, 특정 조건에서 깨어나야 할 경우, 해당 조건을 설정하고 상태를 3번으로 변경한다. 대표적인 경우가 Mutex나 I/O 동작과 같은 경우이다.
4. `TASK_STOPPED`: Process 실행이 중단된 상태이다.
<p align="center">

<img src = "https://wiki.kldp.org/pds/ProcessManagement/state_diagram.jpg">

</p> 

5. `TASK_TRACED`: Process가 다른 process에 의해 추적되고 있는 (ptraced) 상태이다.

아래 함수를 사용하면 커널이 특정 process의 상태를 변경할 수 있다.
~~~
set_task_state(task, state);
~~~
아래 함수를 사용하면 현재 process의 state를 변경할 수 있다.
~~~
set_current_state(state);
~~~

***Process Family Tree***

모든 process는 `init` process의 child이다. `init` process의 PID는 1이며, boot process의 마지막 단계에 활성화된다. `init`의 process descriptor는 `init_task`에 정적으로 할당되고 포인터로 접근한다.

각 process는 하나의 parent를 가진다. Process descriptor에서 `struct task_struct *` 타입의 `parent` 변수를 통해, 해당 process의 parent process에 접근할 수 있다.
~~~
struct task_struct *my_parent = current->parent;
~~~

 Process는 또한 0 이상의 child를 가질 수 있다. `struct task_struct *` 타입을 각 원소로 가지는 list인 `children` 변수를 통해, 해당 process의 각 child process에 접근할 수 있다.
 ~~~
 struct task_struct *task;
 struct list_head *list;

list_for_each(list, &current->children) {
	task = list_entry(list, struct task_struct, sibling);
	/* task now points to one of current's children */
}
~~~

같은 parent process의 직속 children을 sibling이라고 하며, process descriptor들이 circular doubly linked list 구조이기 때문에 `next`, `prev`와 같은 포인터로 쉽게 접근할 수 있다. `next_task(task)`, `prev_task(task)`와 같은 매크로가 구현되어 있다.

## Process Creation
Process의 생성과 실행은 `fork()`와 `exec()` system call로 수행된다. `Fork()`를 호출한 process는 parent process가 되고, 이로 인해 생성된 새로운 process는 child process가 된다. 그 뒤 `exec()`의 호출을 통해, 새로운 process의 이미지를 address space에 로드하여 child process를 수행한다.
<br></br>
**<Fork()>**

`fork() -> clone() -> do_fork() -> copy_process(), get_pid()`
1. 우선 `fork()`에서 parent process와 child process 간에 공유할 자원들을 flag로 설정한다.
2. 해당 flag들을 parameter로 포함시켜 `clone()`을 호출한다.
3. `Clone()`에서는 `do_fork()`를 호출하고, 여기서 `copy_process()`를 호출하여 parent process descriptor의 복제를 통해 child process descriptor를 초기화한다. 또한 process address space의 복제를 통해 child process space를 만든다.
4. `Copy_process()`가 완료되면 `get_pid()`를 통해 child process의 PID를 내부적으로 저장한다.
- 실제 코드: [<kernel/fork.c>](https://github.com/torvalds/linux/blob/f8788d86ab28f61f7b46eb6be375f8a726783636/kernel/fork.c#L2403)

`Fork()`의 반환값은 process에 따라 다르다. 만약 parent process의 코드라면 `fork()`가 완료된 후 생성된 child process의 PID를 반환받는다. 만약 child process의 코드라면 0을 받환받는다. 만약 음수값이 반환된다면 `fork()`가 제대로 이루어지지 않았음을 의미한다.

`Fork()`가 완료되면 parent와 child process는 그 다음 코드부터 동시에 수행하게 된다. 

**예시)**
<p align="center">

<img src = "https://www.csl.mtu.edu/cs4411.ck/www/NOTES/process/fork/fork-4.jpg">

</p> 

**참고)** `Fork()`, `Vfork()`, Copy_on_Write

이전의 `fork()`: Parent process의 자원까지 모두 복사하여 새로운 process 생성까지 시간이 오래 걸린다.

`Vfork()`: Parent process와 child process는 자원을 공유한다. 자원을 복사하는 오버헤드는 감소하였으나 race condition을 막기 위해 부모가 블락되었고, 부모를 깨우기 위한 별도의 코드가 필요하다.

현재의 `fork()`: 이전의 fork에 **copy_on_write** 방식을 도입하였다. Parent process와 child process는 자원을 공유하다가, 새로운 데이터가 쓰여질 경우에만 복사를 실시하여 각자의 자원을 가지도록 한다.
<br></br>
**<Exec()>**

`Exec()`을 호출하면 현재 process의 실행을 다른 process에게 넘겨줄 수 있으므로, 이를 통해 child process를 수행한다. 만약 `fork()`가 수행된 후 child process의 코드에서 바로 `exec()`을 호출하면, parent process보다도 child process를 먼저 실행하여 자원 복사 오버헤드를 더욱 줄일 수 있다.


## Thread Creation

Thread는 특정 process와의 자원 공유를 통해 concurrent programming이 가능하도록 지원한다.

Microsoft Windows, Sum Solaris: 커널에서 thread 관련 함수를 명시적으로 사용한다.

Linux: Thread는 process와 동일하게 생성, 실행, 종료되며, 커널에서 thread를 명시적으로 지원하지 않는다.

Linux에서 thread는 **process와 동일**하게 취급되며, 단지 다른 process와 **자원을 공유할 뿐**이다. 따라서 생성 시 `fork()`에서 `clone()`이 수행되기 전에, `CLONE_FILES`(open files), `CLONE_FS`(file system information), `CLONE_SIGHAND`(signal handlers), `CLONE_VM`(address space) 등의 flag를 설정하여 전달한다. Parent process와 해당 자원들을 공유하는 child process가 곧 thread가 되며, 이때 parent process도 하나의 thread가 된다.

Thread의 실행 역시 process와 마찬가지로, `fork()` 완료 후 `exec()` 호출을 통해 수행된다.

## Process Termination
**<Exit()>**

Process 종료는 두가지 경우로 발생한다

Self-induced: 자신의 process를 마치고 `exit()` system call을 호출한다.

Involuntarily: 특정 신호를 받거나 exception, error 등이 발생하여 강제로 process가 종료된다.

두 경우 모두 `do_exit()` 함수를 호출하여 다음을 수행한다.
1. process의 모든 flag와 사용하던 자원을 해제한다.
2. Parent process에 신호를 보내 종료함을 알린다. 만약 parent process가 먼저 종료되었다면  `forget_original_parent()`, `find_new_reaper()`를 호출하여 현재 thread 그룹에서 새로운 parent process를 찾는다. 적절한 parent process를 찾지 못하면 `init` process의 child process가 된다.
3. 자신의 process 상태를 `EXIT_ZOMBIE`라는 종료 상태로 변경한다.
4. `schedule()`을 호출하여 다른 process로 switch한다. (`Do_exit()`은 반환값이 없다.)
- 실제 코드: [<kernel/exit.c>](https://github.com/torvalds/linux/blob/f8788d86ab28f61f7b46eb6be375f8a726783636/kernel/exit.c#L711)

**<Wait()>**

Parent process는 child process가 끝나기를 기다렸다가 관련된 모든 객체를 해제해야한다. Child process의 상태가 종료 상태로 바뀌면 child process kernel stack, `thread_info` 구조체, `task_struct` 구조체를 해제하고 child process descriptor를 제거하여 완전히 종료시킨다.

<br></br>
#### Image Reference
[그림 1. Process Descriptor] https://img1.daumcdn.net/thumb/R800x0/?scode=mtistory2&fname=https%3A%2F%2Ft1.daumcdn.net%2Fcfile%2Ftistory%2F9986143359D9CE5D14

[그림 2. Process State] https://wiki.kldp.org/pds/ProcessManagement/state_diagram.jpg

[그림 3. Fork()] https://www.csl.mtu.edu/cs4411.ck/www/NOTES/process/fork/fork-4.jpg
