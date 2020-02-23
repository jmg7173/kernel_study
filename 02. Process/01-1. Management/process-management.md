# Process Management
## 목차
- [Process 란](#the-process)
- [Process 세부 정보](#process-descriptor)
(PID, Process State, Process Family Tree)
- [Process 생성](#process-creation)
(Process 실행)
- [Thread 생성](#thread-creation)
- [Process 종료](#process-termination)

## The Process
**Process**의 정의: 실행중인 코드, 실행중인 프로그램
Process의 의미: 실행중인 프로그램 + 관련된 자원들, 실행중인 코드의 결과물

Process는 생성되었을 때 life cycle이 시작된다.

Linux에서 **Thread**는 특별한 종류의 process이다. (Thread 관련 자세한 내용은 [Thread 생성](#thread-creation) 항목으로)
Process는 하나 이상의 thread로 이루어져 있으며, 커널은 process가 아닌 thread 단위로 스케줄링을 진행한다.

참고) **Task**는 작업의 최소 단위를 일컫는 용어이다. 즉 task는 process를 지칭할 수도, thread를 지칭할 수도 있다. 

## Process Descriptor
커널은 circular doubly linked list 구조인 `task list`에 process 세부 정보들을 저장한다. `Task list`의 각 원소들은 `struct task_struct` 타입의 구조체이며, 이들을 **Process Descriptor**라고 한다.
<br></br>
**<위치>**

각 process는 커널 영역에서 사용할 스택인 process kernel stack을 가진다. 이 스택의 최상단 주소에 `struct thread_info` 구조체가 위치된다.

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

실제 구조체 코드: [<linux/sched.h>](https://github.com/torvalds/linux/blob/master/include/linux/sched.h)

***PID***

각 Process를 구분하기 위해 `pid_t` 타입의 `pid` 변수를 가진다. `pid_t` 타입은 process 번호를 저장한다는 의미를 가지며, `pid`는 integer 값으로 표현된다. `PID_MAX_DEFAULT`는 32,768로 정해져있다. (`32bit short int`)

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
