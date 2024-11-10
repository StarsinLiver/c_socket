## epoll 의 이해와 활용

사실 select 는 오래 전에 개발된 멀티플렉싱 기법이다. 때문에 이를 이용하면 아무리 프로그램의 성능을 최적화 시킨다 해도, 허용할 수 있는 동시접속자의 수가 백을 넘기 힘들다. 이러한 select 방식은 웹 기반의 서버개발이 주를 이루는 오늘날의 개발 환경에서는 적절치 않다. 따라서 이에 대한 대안으로 리눅스 영역에서 주로 활용되는 epoll 에 대해 공부하고자 한다.

## select 기반의 IO 멀티플렉싱이 느린 이유

Chapter 12 에서 select 기반의 멀티플렉싱 서버를 구현해 본 경험이 있기 때문에 코드상에서의 불합리한 점을 쉽게 할 수 있는데, 가장 큰 두 가지는 다음과 같다.

```
1. select 함수호출 이후에 항상 등장하는, 모든 파일 디스크립터를 대상으로 하는 반복문
2. select 함수를 호출할 때마다 인자로 매번 전달해야 하는 관찰대상에 대한 정보들
```

위의 두 가지는 Chapter 12 에서 소개한 예제 echo_selectserv.c 의 45 , 49 그리고 54 행에서 확인 가능하다. select 함수가 호출되고 나면, 상태변화가 발생한 파일 디스크립터만 따로 묶이는 것이 아니라, 관찰대상을 묶어서 인자로 전달한 fd_set형 변수의 변화를 통해서 상태변화가 발생한 파일 디스크립터를 구분하기 때문에 모든 파일 디스크립터를 대상으로 하는 반복문의 삽입은 어쩔 수 없는 일이다. 뿐만 아니라 관찰 대상을 묶어놓은 fd_set형 변수에 변화가 생기기 때문에 select 함수의 호출이전에 원본을 복사해두고 select 함수를 호출할 때마다 새롭게 관찰대상의 정보를 전달해야 한다.

그러면 우리는 어떤 것이 성능향상에 더 큰 걸림돌이라고 생각하는가? 그러니깐 select 함수호출 이후에 항상 등장하는 모든 파일 디스크립터 대상의 반복문이 더 큰 걸림돌이라고 생각하는가? 아니면 매번 전달해야 하는 관찰대상에 대한 정보들이 더 큰 걸림돌이라고 생각하는가?
코드만 놓고 보면 반복문이라 생각하기 쉽다. 그러나 반복문보다는 매번 전달해야 하는 관찰대상에 대한 정보들이 더 큰 걸림돌이다. 이는 다음을 뜻하는 것이기 때문이다.

"select 함수를 호출할 때마다 관찰대상에 대한 정보를 매번 운영체제에게 전달해야 한다."

응용 프로그램상에서 운영체제에게 데이터를 전달하는 것은 프로그램에 많은 부담이 따르는 일이다. 그리고 이는 코드의 개선을 통해서 덜 수 있는 유형의 부담이 아니기 때문에 성능에 치명적인 약점이 될 수 있다.

"그런데 왜 관찰대상에 대한 정보를 왜 운영체제에게 전달해야 하는가?"

함수 중에는 운영체제의 도움 없이 기능을 완성하는 함수가 있고, 운영체제의 도움이 절대적으로 필요한 함수가 있다. 예를 들어서 사칙연산과 관련된 함수를 우리가 정의했다고 가정해보자. 그렇다면 운영체제의 도움을 필요치 않는다. 그러나 select 함수는 파일 디스크립터 ,정확히 말하면 소켓의 변화를 관찰하는 함수이다. 그런데 소켓은 운영체제에 의해 관리되는 대상이 아닌가? 때문에 select 하뭇는 절대적으로 운영체제에 의해 기능이 완성되는 함수이다. 따라서 이러한 select 함수의 단점은 다음과 같은 방식으로 해결해야 한다.

"운영체제에게 관찰대상에 대한 정보를 딱 한번만 알려주고서, 관찰대상의 범위 또는 내용에 변경이 있을때 변경 사항만 알려주도록 하자."

이렇게 되면 select 함수를 호출할 때마다 관찰대상에 대한 정보를 매번 운영체제에게 전달할 필요가 없다. 단, 이는 운영체제가 이러한 방식에 동의할 경우에나 가능한 일이다. (이러한 방식을 지원할 경우에나 가능한 일이다.) 때문에 운영체제 별로 지원여부도 다르고 지원방식에도 차이가 있다. 참고로 리눅스에서 지원하는 방식을 가리켜 epoll 이라 하고 윈도우에서는 IOCP 라한다.

## select 이거 필요 없는것인가? 아니다! 장점이 있다!

하지만 select 함수도 잘 알고 있어야 한다. 이번 챕터에서 설명하는 epoll 방식은 리눅스에서만 지원되는 방식이다. 이렇듯 개선된 IO 멀티플렉싱 모델은 운영체제 별로 호환되지 않느다.
반면 select 함수는 대부분의 운영체제에서 지원을 한다. 따라서 다음 두가지 유형의 조건이 만족 또는 요구되는 상황이라면 리눅스에서 운영할 서버라 할지라도 굳이 epoll 을 고집할 필요가 없다.

```
1. 서버의 접속자 수가 많지 않다.
2. 다양한 운영체제에서 운영이 가능해야 한다.
```

이렇듯 모든 상황에서 절대 우위를 점하는 구현모델은 존재하지 않는다. 그러니 우리는 하나닁 모델을 고집하기 보다는 모델 별 장단점을 정확히 이해하고 적절히 적용할 수 있어야 한다.

## epoll 의 구현에 필요한 함수와 구조체

select 함수의 단점을 극복한 epoll 에는 다음의 장점이 있다. 이는 앞서말한 select함수의 단점에 상반된 특징이기도 한다.

```
1. 상태변화의 확인을 위한, 전체 파일 디스크립터를 대상으로 하는 반복문이 필요 없다.
2. select 함수에 대응하는 epoll_wait 함수호출시, 관찰대상의 정보를 매번 전달할 필요가 없다.
```

자! 그럼 epoll 기반의 서버 구현에 필요한 세 가지 함수를 소개할 것이다.

```
1. epoll_create   : epoll 파일 디스크립터 저장소 생성
2. epoll_ctl      : 저장소에 파일 디스크립터 등록 및 삭제
3. epoll_wait     : select 함수와 마찬가지로 파일 디스크립터의 변화를 대기한다.
```

select 방식에서는 관찰대상인 파일 디스크립터의 저장을 위해서 fd_set 형 변수를 직접 선언했었다. 하지만 epoll 방식에서는 관찰대상인 파일 디스크립터의 저장을 운영체제가 담당한다. 때문에 파일 디스크립터의 저장을 위한 저장소의 생성을 운영체제에게 요청해야 하는데 이 때 사용되는 함수가 epoll_create 이다.

그리고 관찰대상인 파일 디스크립터의 추가, 삭제를 위해서 select 방식에서는 FD_SET , FD_CLR 함수를 사용하지만, epoll 방식에서는 epoll_ctl 함수를 통해서 운영체제에게 요청하는 방식으로 이뤄진다.

마지막으로 select 방식에서는 파일 디스크립터의 변화를 대기하기 위해서 select 함수를 호출하는 반면, epoll 에서는 epoll_wait 함수를 호출한다. 그런데 이것이 끝이 아니다. select 방식에서는 select 함수호출 시 전달한 fd_set 형 변수의 변화를 통해서 관찰대상의 상태변화를 확인하지만, epoll 방식에서는 다음 구조체 epoll_event를 기반으로 상태변화가 발생한 파일 디스크립터가 별도로 묶인다.

```c
typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;

struct epoll_event
{
  uint32_t events;	/* Epoll events */
  epoll_data_t data;	/* User data variable */
} __EPOLL_PACKED;
```

위의 구조체 epoll_event 기반의 배열을 넉넉한 길이로 선언해서 epoll_wait 함수호출 시 인자로 전달하면, 상태변화가 발생한 파일 디스크립터의 정보가 이 배열에 별도로 묶이기 때문에 select 함수에서 보인, 전체 파일 디스크립터를 대상으로 하는 반복문은 불필요하다. 이렇게 해서 간단히 epoll 방식에서 사용되는 함수와 구조체에 대해서 살펴보았다. 사실 epoll 은 select 방식에 대한 경험이 있으면 어렵지 않게 적용 가능하다.

## epoll_create

참고로 epol 은 리눅스 커널(운영체제의 핵심모듈) 버전 2.5.44에서부터 소개되기 시작하였다. 따라서 epoll의 적용을 위해서는 리눅스 커널의 버전을 확인할 필요가 있다. 하지만 우리가 사용하는 리눅스의 커널은 대부분 2.6 이상일테니 이 부분은 크게 신경쓰지 않아도 된다. 그래도 다음 명령문을 통해 리눅스 커널의 버전을 확인해보자

```
cat /proc/sys/kernel/osrelease

[root@localhost kernel]# cat osrelease
4.18.0-80.el8.x86_64
```

그럼 epoll_create 함수를 살펴보자

```c
#include <sys/epoll.h>

int epoll_create(int size); // 성공 시 epoll 파일 디스크립터 , 실패 시 -1 반환

// size : epoll 인스턴스의 크기 정보
```

epoll_create 함수호출 시 생성되는 파일 디스크립터의 저장소를 가리켜 'epoll 인스턴스'라 한다. 그러나 변형되어서 다양하게 불리고 있으니 약간의 주의가 필요하다. 그리고 매개변수 size 를 통해서 전달되는 값은 epoll 인스턴스의 크기를 결정하는 정보로 사용된다. 하지만 이 값은 단지 운영체제에 전달하는 힌트에 지나지 않는다.
즉, 인자로 전달된 크기의 epoll 인스턴스가 생성되는 것이 아니라, epoll 인스턴스의 크기를 결정하는데 있어서 참고로만 사용된다.

리눅스 커널 2.6.8 이후부터 epoll_create 함수의 매개변수 size 는 완전히 무시된다. 커널 내에서 epoll 인스턴스의 크기를 상황에 맞게 적절히 늘리기도 하고 줄이기도 한다.

그리고 epoll_create 함수호출에 의해서 생성되는 리소스는 소켓과 마찬가지로 운영체제에 의해서 관리가 된다. 따라서 이 함수는 소켓이 생성될 때와 마찬가지로 파일 디스크립터를 반환한다. 즉, 이 함수가 반환하는 파일 디스크립터는 epoll 인스턴스를 구분하는 목적으로 사용이 되며, 소멸 시에는 다른 파일 디스크립터들과 마찬가지로 close 함수호출을 통한 종료의 과정을 거칠 필요가 있다.

## epoll_ctl

epoll 인스턴스 생성 후에는 이곳에 관찰대상이 되는 파일 디스크립터를 등록해야 하는데, 이때 사용하는 함수가 epoll_ctl 이다.

```c
#include <sys/epoll.h>

int epoll_ctl(int epfd , int op, int fd , struct epoll_event *event); // 성공 시 0 , 실패 시 -1 반환

// epfd : 관찰대상을 등록할 epoll 인스턴스의 파일 디스크립터
// op : 관찰대상의 추가, 삭제 또는 변경여부 지정
// fd : 등록할 관찰대상의 파일 디스크립터
// event : 관찰대상의 관찰 이벤트 유형
```

epoll 의 다른 함수들보다 다소 복잡해 보인다. 하지만 문장을 통해서 쉽게 이해가 가능하다. 예를 들어서 다음의 형태로 epoll_ctl 함수가 호출되었다면

```c
epoll_ctl(A, EPOLL_CTL_ADD , B , C);
```

두번째 인자인 EPOLL_CTL_ADD 는 '추가'를 의미한다. 따라서 위 문장은 다음의 의미를 갖는다.

"epoll 인스턴스 A에 파일 디스크립터 B를 등록하되, C를 통해 전달된 이벤트의 관찰을 목적으로 등록을 진행한다."

한문장 더 소개하겠다.

```c
epoll_ctl(A , EPOLL_CTL_DEL , B , NULL);
```

위 문장의 두번째 인자인 EPOLL_CTL_DEL 은 '삭제'로 다음을 의미한다.

"epoll 인스턴스 A에서 파일 디스크립터 B를 삭제한다."

위 문장에서 보이듯이 관찰대상에서 삭제할 때에는 관찰유형, 즉, 이벤트 정보가 불필요하기 때문에 네번째 인자로 NULL 이 전달되었다. 그럼 먼저 epoll_ctl 의 두번째 인자로 전달가능한 상수와 그 의미를 정리해보겠다.

```
EPOLL_CTL_ADD : 파일 디스크립터를 epoll 인스턴스에 등록
EPOLL_CTL_DEL : 파일 디스크립터를 epoll 인스턴스에서 삭제
EPOLL_CTL_MOD : 등록된 파일 디스크립터의 이벤트 발생상황을 변경
```

이 중에서 EPOLL_CTL_MOD는 잠시 후에 알게 될 것이다. 그리고 앞서 두번째 인자 EPOLL_CTL_DEL 은 네번째 인자가 NULL 이 들어가는게 타당하다. 그러나 리눅스 2.6.9 에서는 NULL 을 허용하지 않았다. 비록 전달되는 값은 그냥 무시되지만 그래도 epoll_event 구조체 변수의 주소값을 전달해아만 했다.

그럼 이제 구조체 epoll_event 의 포인터를 설명하겠다.

"어라? 구조체 epoll_event 는 상태변화가 발생한(이벤트가 발생한) 파일 디스크립터를 묶는 용도로 사용된다고 하지 않았어요?"

물론 그렇다! 잠시 후에 소개하겠지만, 앞서 말했듯이 구조체 epoll_event는 이벤트가 발생한 파일 디스크립터를 묶는 용도로 사용된다. 하지만 파일 디스크립터를 epoll 인스턴스에 등록할 때, 이벤트의 유형을 등록하는 용도로도 사용된다. 그런데 구조체 epoll_event 가 눈에 확 들어오게 정의되어 있는것은 아니니 문장을 통해서 이 구조체가 epoll_ctl 함수에 어떻게 활용되는지 설명하겠다.

```c
struct epoll_event event;
event.events = EPOLLIN;
event.data.fd = sockfd;
epoll_ctl(epdf , EPOLL_CTL_ADD , sockfd , &event);
```

위 코드는 epoll 인스턴스인 epfd 에 sockfd 를 틍록하되 수신할 데이터가 존재하는 상황에서 이벤트가 발생하도록 등록하는 방법을 보이고 있다. 그럼 이벤에는 epoll_event 의 멤버인 events 에 저장가능한 상수와 이벤트의 유형에 대해 정리해보겠다.

| 이벤트 유형  |                                                                                       내용                                                                                       |
| :----------: | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------: |
|   EPOLLIN    |                                                                          수신할 데이터가 존재하는 상황                                                                           |
|   EPOLLOUT   |                                                              출력버퍼가 비워져서 당장 데이터를 전송할 수 있는 상황                                                               |
|   EPOLLPRI   |                                                                             OOB 데이터가 수신된 상황                                                                             |
|  EPOLLRDHUP  |                                         연결이 종료되거나 Half-close 가 진행된 상황, 이는 엣지 트리거 방식에서 유용하게 사용될 수 있다,                                          |
|   EPOLLERR   |                                                                                에러가 발생한 상황                                                                                |
|   EPOLLET    |                                                                 이벤트의 감지를 엣지 트리거 방식으로 동작시킨다.                                                                 |
| EPOLLONESHOT | 이벤트가 한번 감지되면 해당 파일 디스크립터에서는 더 이상 이벤트를 발생시키지 않는다. 따라서 epoll_ctl 함수의 두 번째 인자로 EPOLL_CTL_MOD 를 전달해서 이벤트를 재설정해야 한다. |

위 상수들은 비트 OR 연산자를 이용해서 둘 이상을 함께 등록할 수 있다. 그리고 이 중에서 '엣지 트리거'라는 말이 나오는데 이는 잠시 후에 별도로 설명을 하니, 일단은 EPOLLIN 하나만 기억하고 있기를 바란다.

## epoll_wait

이제 마지막으로 select 함수에 해당하는 epoll_wait 함수를 소개할 차례이다. 기본적으로 이 함수가 epoll 관련 함수 중에서 가장 마지막에 호출된다.

```c
#include <sys/epoll.h>

int epoll_wait(int epfd , strcut epoll_event * events , int maxevents , int timeout); // 성공시 이벤트가 발생한 파일 디스크립터의 수, 실패 시 -1 반환

// epdf : 이벤트 발생의 관찰영역인 epoll 인스턴스의 파일 디스크립터
// events : 이벤트가 발생한 파일 디스크립터가 채워질 버퍼의 주소 값
// amxevents : 두번째 인자로 전달된 주소 값의 버퍼에 등록 가능한 최대 이벤트 수
// timeout : 1/1000초 단위의 대기시간, -1 전달 시, 이벤트가 발생할 때까지 무한 대기
```

이 함수의 호출방식은 다음과 같다. 여기서는 두번째 인자로 전달되는 주소 값의 버퍼를 동적으로 할당해야 한다는 점만 주목하면 된다.

```c
int event_cnt;
struct epoll_event * ep_events;
ep_events = malloc(sizeof(struct epoll_event) * EPOLL_SIZE); // EPOLL_SIZE 는 매크로 값
event_cnt = epoll_wait(epfd , ep_events , EPOLL_SIZE , -1);
```

함수 호출 후에는 이벤트가 발생한 파일 디스크립터의 수가 반환되고, 두 번째 인자로 전달된 주소 값의 버퍼에는 이벤트가 발생한 파일 디스크립터의 정보가 별도로 묶이기 때문에 SELECT 방식에서 보인 전체 파일디스크립터를 대상으로 하는 반복문의 삽입이 불필요하다.

## epoll 기반의 에코 서버

이로써 epoll 기반의 서버구현에 필요한 이론적인 설명을 모두 마쳤다. 따라서 epoll 기반의 에코서버를 제작해 보자.

```c
//
// Created by root on 24. 11. 4.
// echo_epollserv.c
//

#include <stdio.h>
#include <sys/epoll.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 100
#define EPOLL_SIZE 50

void error_handling(char *message);

int main(int argc, char *argv[]) {
    int serv_sock, clnt_sock;
    struct sockaddr_in serv_adr, clnt_adr;
    socklen_t adr_sz;
    int str_len, i;
    char buf[BUF_SIZE];

    struct epoll_event *ep_events;
    struct epoll_event event;
    int epfd , event_cnt;
    if (argc != 2) {
        printf("Usage : %s <Port> \n", argv[0]);
        exit(1);
    }

    serv_sock = socket(PF_INET, SOCK_STREAM, 0);
    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_adr.sin_port = htons(atoi(argv[1]));

    if (bind(serv_sock, (struct sockaddr *) &serv_adr, sizeof(serv_adr)) == -1)
        error_handling("bind() error");

    if (listen(serv_sock, 5) == -1)
        error_handling("listen()error");

    epfd = epoll_create(EPOLL_SIZE); // epoll 인스턴스 생성
    ep_events = malloc(sizeof(struct epoll_event) * EPOLL_SIZE);

    event.events = EPOLLIN;
    event.data.fd = serv_sock;
    epoll_ctl(epfd , EPOLL_CTL_ADD, serv_sock , &event);

    while(1) {
        event_cnt = epoll_wait(epfd , ep_events , EPOLL_SIZE , -1);
        if(event_cnt == -1) {
            puts("epoll_wait() error");
            break;
        }

        for(i = 0; i < event_cnt; i++) {
            if(ep_events[i].data.fd == serv_sock) {
                adr_sz = sizeof(clnt_adr);
                clnt_sock = accept(serv_sock , (struct sockaddr*) & clnt_adr , &adr_sz);
                event.events = EPOLLIN;
                event.data.fd = clnt_sock;
                epoll_ctl(epfd , EPOLL_CTL_ADD , clnt_sock , &event);
                printf("connect client %d \n" , clnt_sock);
            } else {
                str_len = read(ep_events[i].data.fd , buf , BUF_SIZE);
                if(str_len == 0) {
                    epoll_ctl(epfd , EPOLL_CTL_DEL, ep_events[i].data.fd , NULL);
                    close(ep_events[i].data.fd);
                    printf("close client : %d \n" , ep_events[i].data.fd);
                } else {
                    write(ep_events[i].data.fd , buf , str_len); // echo
                }
            }
        }
    }

    close(serv_sock);
    close(clnt_sock);
    return 0;
}

void error_handling(char *message) {
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```
