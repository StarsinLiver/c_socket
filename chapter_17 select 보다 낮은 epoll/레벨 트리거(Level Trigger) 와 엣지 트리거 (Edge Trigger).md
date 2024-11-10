## 레벨 트리거(Level Trigger) 와 엣지 트리거 (Edge Trigger)

epoll 을 알면서 레벨 트리거 방식과 엣지 트리거 방식의 차이를 모르는 경우가 많다. 하지만 이를 알아야 epoll 을 완전히 이해하는 것이다.

## 레벨 트리거와 엣지 트리거 차이는 이벤트가 발생하는 시점에 있다.

우선 간단히 레벨 트리거와 엣지 트리거의 이해를 위해서 예를 들겠다. 먼저 다음의 대화내용을 관찰하자

```
// 레벨 트리거
아들 : 엄마 세벳돈으로 5000원을 받았어요
엄마 : 훌륭하구나!

아들 : 엄마 옆집 숙희가 떡볶이 사달래서 사줬더니 2000원 남았어요
엄마 : 장하다 우리아들

아들 : 엄마 변신가면 샀더니 500원 남았어요
엄마 : 그래 용돈 다 쓰면 굶으면 된다!

아들 : 엄마 여전히 500원 갖고 있어요 굶을 순 없잖아요
엄마 : 그래 매우 현명하구나!

아들 : 엄마 여전히 500원 갖고 있어요 끝까지 지켜야지요.
엄마 : 그래 힘내거라!
```

위 대화에서 보면 아들은 자신의 수중에 돈이 들어올 때부터 돈이 남아있는 동안 계속해서 엄마에게 보고를 한다. 이것이 바로 레벨 트리거의 원리이다. 위 대화에서 아들을 입력버퍼, 세뱃돈을 입력 데이터 그리고 아들의 보고를 이벤트로 바꿔서 이해하면 레벨 트리거의 특성이 발견된다. 이를 정리하면 다음과 같다.

"레벨 트리거 방식에서는 입력버퍼에 데이터가 남아있는 동안에 계속해서 이벤트가 등록된다."

예를 들어서 서버의 입력버퍼로 50바이트의 데이터가 수신되면 일단 서버 측 운영체제는 이를 이벤트로 등록한다.(변화가 발생한 파일 디스크립터로 등록한다.) 그런데 서버 프로그램에서 20바이트를 수신해서 입력버퍼에 30바이트만 남는다면 이 상황 역시 이벤트로 등록이 된다. 이렇듯 레벨 트리거 방식에서는 입력버퍼에 데이터가 남아있기만 해도 이 상황을 이벤트로 등록한다.

그럼 이번에는 다음 대화를 통해서 엣지 트리거의 이벤트 특성을 이해해보자.

```
// 엣지 트리거
아들 : 엄마 세뱃돈으로 5000원을 받았어요.
엄마 : 음 다음엔 더 노력하거라.

아들 : ......
엄마 : 말 안하니? 돈 으쨋어
```

위 대화에서 보이듯이 엣지 트리거는 입력버퍼로 데이터가 수신된 상황에서 딱! 한번만 이벤트가 등록된다. 때문에 입력버퍼에 데이터가 남아있다고 해서 이벤트를 추가로 등록하지 않는다.

## 레벨 트리거의 이벤트 특성 파악하기

그럼 먼저 레벨 트리거의 이벤트 등록방식을 코드상에서 확인해보자 다음은 앞서 소개한 예제 echo_epollserv.c 를 조금 수정한 것이다.

```c
// echo_EPLTserv.c
//
// Created by root on 24. 11. 4.
//

#include <stdio.h>
#include <sys/epoll.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 4 // <------------------------  수정
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

    epfd = epoll_create(EPOLL_SIZE);
    ep_events = malloc(sizeof(struct epoll_event) * EPOLL_SIZE);

    event.events = EPOLLIN;
    event.data.fd = serv_sock;
    epoll_ctl(epfd , EPOLL_CTL_ADD, serv_sock , &event);

    while(1) {
        event_cnt = epoll_wait(epfd , ep_events , EPOLL_SIZE , -1);
        if(event_cnt == -1) {
            puts("epoll_wait() error!");
            break;
        }

        puts("return epoll_wait"); // <------------------------  추가
        for(i = 0; i < event_cnt; i++) {
            if(ep_events[i].data.fd == serv_sock) {
                adr_sz = sizeof(clnt_adr);
                clnt_sock = accept(serv_sock , (struct sockaddr*) &clnt_adr , &adr_sz);
                event.events = EPOLLIN;
                event.data.fd = clnt_sock;
                epoll_ctl(epfd , EPOLL_CTL_ADD , clnt_sock , &event);
                printf("connected client %d \n" , clnt_sock);
            } else {
                str_len = read(ep_events[i].data.fd , buf , BUF_SIZE);
                if(str_len == 0) {
                    epoll_ctl(epfd , EPOLL_CTL_DEL , ep_events[i].data.fd, NULL);
                    close(ep_events[i].data.fd);
                    printf("closed client %d \n" , ep_events[i].data.fd);
                } else {
                    write(ep_events[i].data.fd , buf, str_len); // echo
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

```
[root@localhost chater17]# ./echo_EPLTserv 9190
return epoll_wait
connected client 5
return epoll_wait
return epoll_wait
return epoll_wait
return epoll_wait
return epoll_wait
return epoll_wait

connected client 6
return epoll_wait
return epoll_wait
return epoll_wait
return epoll_wait

closed client 6
closed client 5
```

버퍼의 크기를 줄인 이유는 입력버퍼에 수신된 데이터를 한번에 읽어 들이지 못하게 하기 위함이다. 즉, read 함수호출 이후에도 입력버퍼에는 여전히 읽어 들일 데이터가 존재할 것이고, 이로 인해 새로운 이벤트가 등록되어서 epoll_wait 함수가 반환을 한다면, 문자열 "return epoll_wait" 이 반복 출력될 것이다. 레벨 트리거의 동작방식이 필자가 설명한 것과 일치한다면 말이다.

실행결과는 클라이언트로부터 메시지를 한번 수신할 때마다 이벤트 등록이 여러번 이뤄지고, 이로인해 epoll_wait 함수가 다수 호출됨을 보이고 있다.
그럼 이번에는 간단히 위 예제를 엣지 트리거 방식으로 변경해 보겠다. 사실 엣지 트리거 모델로의 변경을 위해서는 추가로 손 볼 부분이 조금 있다. 그러나 최소한의 수정으로 엣지 트리거 모델의 이벤트 등록방식만 간단히 확인해 보고자 한다.이를 위해 EPOLLET 를 추가한다. 그러면 다음 사실을 확인할 수 있다.

"클라이언트로부터 데이터가 수신될 때, 딱 한번 문자열 "return epoll_wait" 가 출력된다. 그리고 이는 이벤트가 딱 한번 등록됨을 의미한다."

그런데 위의 사실은 확인 가능하지만, 클라이언트의 실행결과에는 문제가 발생한다.

## select 모델은 레벨 트리거 일까 엣지 트리거 일까

select 모델은 레벨 트기러 방식으로 동작한다. 즉, 입력버퍼에 데이터가 남아있으면 무조건 이벤트가 등록된다.

## 엣지 트리거 기반의 서버 구현을 위해서 알아야 할 것 두 가지!

이어서 엣지 트리거 기반의 서버 구현 방법에 대해 설명할 텐데 이에 앞서 다음 두 가지를 설명하고자 한다. 이는 엣지 트리거 구현에 있어서 필요한 내용이다.

```
1. 변수 errno 을 이용한 오류의 원인을 확인하는 방법
2. 넌-블로킹(Non-blocking) IO 를 위한 소켓의 특성을 변경하는 방법
```

일반적으로 리눅스에서 제공하는 소켓관련 함수는 -1 반환함으로써 오류의 발생을 알린다. 따라서 오류가 발생했을음 인식할 수 있으나, 이것만으로는 오류의 원인을 정확히 확인할 수 없다. 때문에 리눅스에서는 오류발생 시 추가적인 정보의 제공을 위해서 다음의 변수를 전역으로 선언해 놓고 있다.

```c
int errno;
```

그리고 이 변수의 접근을 위해서는 헤더파일 errno.h 를 포함해야 한다. 이 헤더파일에 위 변수 extern 선언이 존재하기 때문이다.
그리고 함수 별로, 오류발생시 변수 errno에 저장되는 값이 다르기 때문에 이 변수에 저장되는 값을 지금 모두 알려고 달려들 필요는 없다. 함수를 공부하면서 필요할 때마다 조금씩 알아가면 되고, 필요할 때마다 참조할 수 있으면 된다.

"read 함수는 입력버퍼가 비어서 더 이상 읽어 들일 데이터가 없을 때 -1 을 반환하고 이 때 errno에는 상수 EAGAIN가 저장된다."

이 변수의 사용 예는 잠시 후 예제를 통해서 확인하기로 하고, 이번에는 소켓을 넌-블로킹 모드로 변경하는 방법을 소개하겠다.

## 소켓 Non-blocking (넌 블로킹)

리눅스에는 파일의 특성을 변경 및 참조하는 다음 함수가 정의되어 있다. (Chapter 13 에 사용한 바가 있다.)

```c
#include <fcntl.h>

int fcntl(int filedes , int cmd , ...); // 성공 시 매개변수 cmd 에 따른 값, 실패 시 -1 반환

// filedes : 특성 변경의 대상이 되는 파일의 파일 디스크립터 전달
// cmd : 함수호출의 목적에 해당하는 정보 전달.
```

위에서 보이듯이 fcntl ㅎ마수는 가변인자의 형태로 정의되어 있다. 그리고 두 번째 인자로 F_GETFL을 전달하면, 첫번째 인자로 전달된 파일 디스크립터에 설정되어 있는 특성정보를 int 형으로 얻을 수 있으며 반대로 F_SETFL을 인자로 전달해서 특성 정보를 변경할 수도 있다.

따라서 파일을(소켓을)넌-블로킹 모드로 변경하기 위해서는 다음 두 문장을 실행하면 된다.

```c
// file (socket) Non blocking mode
int flag = fcntl(fd , F_GETFL , 0); // 파일 디스크립터 특성 정보 얻기
fcntl(fd , F_SETFL , flag | O_NONBLOCK); // 특성 정보 변경
```

첫 번째 문장을 통해서 기존에 설정되어 있던 특성정보를 얻고, 두번째 문장에서 넌 블로킹 입출력을 의미하는 O_NONBLOCK을 더해서 특성을 재설정해주고 있다. 이로써 read & write 함수호출 시에도 데이터의 유무에 상관없이 블로킹이 되지 않는 파일(소켓)이 만들어 진다. 여기서 설명한 fcntl 함수 역시 광범위하게 사용되므로 시스템 프로그래밍을 공부하면서 조금씩 알아가자.

## 엣지 트리거 기반의 에코 서버 구현

먼저 errno 을 이용한 오류의 확인과정이 필요한 이유를 설명하겠다.

"엣지 트리거 방시에서는 데이터가 수신되면 딱 한번 이벤트가 등록된다."

이러한 특성때문에 일단 입력과 관련해서 이벤트가 발생하면 입력버퍼에 저장된 데이터 전부를 읽어 들여야 한다. 따라서 앞서 설명한 다음 내용을 기반으로 입력 버퍼가 비어있는지 확인하는 과정을 거쳐야한다.

"read 함수가 -1 을 반환하고, 변수 errno 에 저장된 값이 EAGAIN이라면 더 이상 읽어 들일 데이터가 존재하지 않는 상황이다."

그럼 소켓을 넌0블로킹 모드로 만드는 이유ㅜ는 어디에 있을까? 엣지트리거 방식의 특성상 블로킹으로 동작하는 read & write 함수의 호출은 서버를 오랜 기간 멈추는 상황으로까지 이어지게 할 수 있다. 때문에 엣지 트리거 방식에서는 반드시 넌-블로킹 소켓을 기반으로 read & write 함수를 호출해야 한다. 그럼 이제 엣지 트리거 방식으로 동작하는 에코 서버를 소개하겠다.

```c
// code
// Created by root on 24. 11. 4.

#include <fcntl.h>
#include <errno.h>
#include <stdio.h>
#include <sys/epoll.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 4
#define EPOLL_SIZE 50

void setnonblockingmode(int fd);

void error_handling(char *message);

int main(int argc, char *argv[]) {
    int serv_sock, clnt_sock;
    struct sockaddr_in serv_adr, clnt_adr;
    socklen_t adr_sz;
    int str_len, i;
    char buf[BUF_SIZE];

    struct epoll_event *ep_events;
    struct epoll_event event;
    int epfd, event_cnt;
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

    setnonblockingmode(serv_sock);
    event.events = EPOLLIN;
    event.data.fd = serv_sock;
    epoll_ctl(epfd, EPOLL_CTL_ADD, serv_sock, &event);

    while (1) {
        event_cnt = epoll_wait(epfd, ep_events, EPOLL_SIZE, -1);
        if (event_cnt == -1) {
            puts("epoll_wait() error");
            break;
        }

        puts("return epoll_wait");
        for (i = 0; i < event_cnt; i++) {
            if (ep_events[i].data.fd == serv_sock) {
                adr_sz = sizeof(clnt_adr);
                clnt_sock = accept(serv_sock, (struct sockaddr *) &clnt_adr, &adr_sz);
                setnonblockingmode(clnt_sock);
                event.events = EPOLLIN | EPOLLET;
                event.data.fd = clnt_sock;
                epoll_ctl(epfd, EPOLL_CTL_ADD, clnt_sock, &event);
                printf("connect client %d \n", clnt_sock);
            }
            else {
                while(1) {
                    str_len = read(ep_events[i].data.fd, buf, BUF_SIZE);
                    if (str_len == 0) {
                        epoll_ctl(epfd, EPOLL_CTL_DEL, ep_events[i].data.fd, NULL);
                        close(ep_events[i].data.fd);
                        printf("close client : %d \n", ep_events[i].data.fd);
                    } else if (str_len == 0) {
                        if(errno == EAGAIN)
                            break;
                    }
                    else {
                        write(ep_events[i].data.fd, buf, str_len); // echo
                    }
                }

            }
        }
    }

    close(serv_sock);
    close(clnt_sock);
    return 0;
}

void setnonblockingmode(int fd) {
    int flag = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flag | O_NONBLOCK);
}

void error_handling(char *message) {
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

```
[root@localhost chater17]# ./echo_EPETserv 9190
return epoll_wait
connect client 5
return epoll_wait
close client : 5
```

## 레벨 트리거와 엣지 트리거 중에 뭐가 더 좋은가요?

엣지트리거 방식을 사용하면 다음과 같은 형태의 구현이 가능하다.

"데이터의 수신과 데이터가 처리되는 시점을 분리할 수 있다!"

이는 단순하지만 엣지 트리거의 장점을 매우 정확하고 강력하게 설명하고 있다.

![alt text](/image/28.png)

위 그림에서 연출한 시나리오는 다음과 같다.

```
1. 서버는 클라이언트 A , B , C로 부터 각각 데이터를 수신한다.
2. 서버는 수신한 데이터를 A,B,C 의 순으로 조합한다.
3. 조합한 데이터는 임의의 호스트에게 전달한다.
```

위의 시나리오를 완성하기 위해서 다음과 같은 형태로만 흐름이 전개되면 서버의 구현은 그리 어렵지 않다.

```
1. 클라이언트 A,B,C 가 순서대로 접속해서 데이터를 순서대로 서버에 전송한다.
2. 데이터를 수신할 클라이언트는 클라이언트 A,B,C에 앞서 먼저 접속을 하고 대기한다.
```

그러나 현실적으로는 다음과 같은 상황이 빈번히 일어난다.

```
1. 클라이언트 C와 B는 서버로 데이터를 전송하고 있는데, A는 아직 연결조차 하지 않은 경우
2. 클라이언트 A,B,C 가 순서에 상관없이 데이터를 서버로 전송하는 경우
3. 서버로 데이터가 전송되고 있는데, 정작 이 데이터를 수신할 클라이언트가 아직 연결되지 않은 경우
```

따라서 입력버퍼에 데이터가 수신된 상황임에도 불구하고 이를 읽어 들이고 처리하는 시점을 서버가 결정할 수 있도록 하는 것은 서버 구현에 엄청난 유연성을 제공한다.

"레벨 트리거는 데이터의 수신과 데이터의 처리를 구분할 수 없나요?"

억지로 하면 된다. 하지만 입력버퍼에 데이터가 수신된 상황에서 이를 읽어 들이지 않으면 (이의 처리를 뒤로 미루면) epoll_wait 함수를 호출할 때마다 이벤트가 발생할 테고, 이로 인해서 발생하는 이벤트의 수가 계속해서 누적될 텐데 이를 감당할 수 없다.

이렇듯 레벨 트리거와 엣지 트리거의 차이점은 서버의 구현모델에서 먼저 이야기해야 한다.

"엣지 트리거가 빠른가요? 그리고 빠르면 얼마나 더 빠른가요?"

구현모델의 특성상 엣지 트리거가 좋은 성능을 발휘할 확률이 상대적으로 높은 것은 사실이다. 하지만 단순히 엣지 트리거를 적용했다고 해서 무조건 빨라지는 것은 아니다.
