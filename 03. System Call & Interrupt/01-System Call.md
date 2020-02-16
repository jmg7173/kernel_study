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

Exception이나 Trap을 제외하고 정상적인 방법으로 Kernel에 진입할 수 있는 유일한 방법이다.