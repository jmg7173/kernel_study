# System Call

OS에는 사용자가 사용할 수 있는 영역(User Space)와 커널에서 사용하는 영역(Kernel Space)로 나뉘어 있다. System Call은 User Space에서 실행되는 Process가 Kernel Space에 접근할 수 있도록 인터페이스를 제공하는 역할을 한다.

User Process에서 System Call을 호출하면 Kernel space로 전환되게 된다. 이를 통해 Kernel 영역에서만 사용가능한 기능이나, Kernel 영역에서만 사용가능한 영역에 접근할 수 있다.

## System Call의 역할

Linux Kernel Development에서는 System Call의 역할을 다음과 같이 정의하고 있다.
1. User Space를 위한 추상화된 Hardware Interface를 제공한다.

예를 들어 read / write의 경우 kernel에서 제공된 시스템 콜이지만 HDD나 SSD에 대한 지식이 없어도 이를 추상화 하여 물리적인 저장장치에 데이터를 쓰고 읽을 수 있도록 도와준다. printf의 경우 printf는 유저가 호출하는 함수이지만, 내부 구현을 살펴보면 write 시스템 콜을 호출해 화면에 문자를 출력할 수 있다.

2. 시스템의 보안과 안정성을 보장한다.

시스템 콜은 User space와 시스템 리소스간의 중개자와 같은 역할을 한다. 따라서, 시스템 콜을 통해 kernel space에 접근하지 않는다면 user application은 함부로 하드웨어를 잘못 사용하거나 다른 프로세스의 리소스를 훔치는 등 시스템에 해를 끼치는 일을 할 수 없다.

3. User space와 시스템 간의 공통된 부분으로, 가상화된 시스템이 프로세스에 제공되도록 한다.

Exception이나 Trap을 제외하고 User Application에서 정상적인 방법으로 Kernel에 진입할 수 있는 유일한 방법이다.

## System Call 만들기
모든 System Call의 return 값은 long이고, 정상 처리 시 주로 0을 return하나 에러 발생 시 음수 값을 리턴하고 이 값이 kernel에 등록되어있다면 perror 함수를 통해 무슨 에러인지 알 수 있다. 대표적인 return 값으로는 `-EINVAL`이 있다.

### Kernel code에 System Call 정의
System Call 호출시 여러 인자를 넘길 수 있는데, 몇 개의 인자를 넘길 수 있는지는 Kernel 컴파일 시 정할 수 있다. 기본 설정은 최대 6개까지이고, 넘기는 갯수에 따라 정의하는 방법이 다르다. 다음은 System Call 정의의 일부이다. Kernel code에 다음과 같이 등록할 System Call에 대해 정의해야 한다. 이 과정은 trap을 통해 system call에 진입하기 위함이다.

```c
SYSCALL_DEFINE0(syscall_name)
{
    ...
}

SYSCALL_DEFINE1(syscall_name, param1 type, param1 name)
{
    ...
}

SYSCALL_DEFINE2(syscall_name, param1 type, param1 name, param2 type, param2 name)
{
    ...
}
...
```

### System Call Table 및 Header에 등록
System Call은 위와 같이 Kernel 코드에 정의하는 것 이외에 system call table에 등록해야하며, linux header중 systemcall에 대한 header에 등록하고자 하는 System Call을 등록해야 사용할 수 있다. 두 항목은 linux source code 중 다음의 위치에 있다.

* System Call Table: `arch/[architecture]/syscalls/syscall_64.tbl`
* System Call Header: `include/linux/syscalls.h`

#### System Call Table

System Call Table에 등록할 때는 다음과 같은 형식으로 등록하게 된다.
```
<system call number> <abi> <name> <entry point>

ex) 0 common read sys_read
```
`abi`의 경우 어느 아키텍쳐에서 동작 가능하게 할 지를 의미하는데, 보통의 경우 common을 사용하며, x86 아키텍쳐를 쓰느냐, arm 아키텍쳐를 쓰느냐에 따라 common외에 어떤 것이 사용가능한지 정해져 있으므로, System Call Table을 수정할 아키텍쳐에 맞춰 System Call Table 파일 상단의 설명을 참조하도록 한다.

`entry point`는 user process에서 system call을 호출할 때 kernel에 어떤 이름으로 등록되어 호출될지를 의미한다. 모든 시스템 콜에 해당되는 것은 아니나, 대부분의 경우 원래 이름에 `sys_`라는 접미어가 붙게 된다. System Call Header에 System Call을 등록할 때는 접미어가 붙은 이름으로 등록되게 된다.

#### System Call Header

System Call Header에 등록할 때는 다음과 가튼 형식으로 등록하게 된다.
```c
asmlinkage long sys_[system call name](...params...);

ex) asmlinkage long sys_read(unsigned int fd, char __user *buf, size_t count);
```

함수 선언에 있어서 특이하게 볼 수 있는 부분은 `asmlinkage`이다. Kernel 코드의 경우 최적화 옵션을 통해 함수의 인자들을 eax, ebx와 같은 register에 저장해 넘김으로써 더 좋은 성능을 얻을 수 있다. 그러나 assembly level에서 문제가 생기는 것을 방지하기 위해 assembly와 link를 할 수 있는 함수들에 `asmlinkage` 키워드를 줌으로써 stack을 이용해 전달한다.

System Call의 return 값은 주로 eax에 저장된다. 만약 `asmlinkage` 키워드를 사용하지 않으면, system call 호출 시 eax에 인자를 저장하는 경우 System Call 동작이 완료된 후 eax에 저장되어있던 원래 값이 의도치않게 바뀔 수 있다. 그러므로 `asmlinkage` 키워드는 System Call 동작에 있어 매우 중요하다.

### `unistd.h`
Header와 Table에 System Call을 등록하고 kernel을 컴파일하면 `include/uapi/asm-generic/unistd.h`에 등록이 되며, 시스템 콜 번호는 `__NR_` 접미어와 함께 macro로 정의되게 된다. 단, system call table에 등록한 system call 번호와 실제로 `unistd.h`에 등록된 번호에는 차이가 있으므로 직접 `unistd.h`에 등록하지 않는것을 권장한다. 다음은 `unistd.h`에 등록된 예시이다.
```c
#define __NR_read 63
__SYSCALL(__NR_read, sys_Read)
```

## System Call Handler

System Call은 assembly level에서 보면 `int 0x80`을 호출함으로써 이뤄진다. 이 명령을 수행하게 되면 interrupt vector의 `0x80`번지를 통해 system call handler로 진입하게 되고(assembly code에서는 system_call()함수로 정의되어있음), `eax`에 저장된 값을 통해 호출하고자 하는 System Call 번호를 얻는다.

System Call Table는 배열로, 각 element에는 System Call들이 있다. System Call number를 통해 System Call Table에 저장되어있는 System Call에 접근할 수 있고, 소프트웨어 Interrupt를 통해 kernel로 trap해 kernel mode로 전환해 system call을 수행한다.

이 내용을 정리하면 다음과 같다.  
<!--![System Call Procedure](./fig/syscall.png)-->

## 주의사항
User Program에서 System Call을 호출하는 경우 parameter를 통해 kernel을 속여 유해한 작업을 수행할 수 도있다. 그래서 System Call에서는 user가 넘기는 parameter들 중에서도 pointer에 대한 확인이 중요하다. Linux Kernel Development에서는 다음의 세가지에 대한 내용을 확인하는 것을 권장한다.

* Pointer는 User Space의 메모리 영역을 가리키며, user process가 kernel 대신 kernel space에 있는 데이터를 읽도록 속일 수 없어야 한다.
* Pointer는 Process의 Address Space 메모리 영역을 가리키며, Process가 다른 User의 데이터를 읽도록 속일 수 없어야 한다.
* 메모리에 대한 작업에 대해 상태가 표시되며, readable / writable / executable로 나뉜다. Process는 메모리 access 제한을 따라야한다.

Kernel에서는 위의 방법을 통해 검사하거나, User 및 Kernel space 간 복사를 통해 사용할 수 있다.

* User space로 write:  
`copy_to_user(dst(kernel mem), src(user mem), size)`
* User space로부터 read:  
`copy_from_user(dst(user mem), src(kernel mem), size)`

## System Call 호출
System Call Table에 등록된 System Call들은 다음과 같이 호출할 수 있다.

```c
#include <linux/unistd.h>
syscall(SYSCALL_NUMBER, args...);
```