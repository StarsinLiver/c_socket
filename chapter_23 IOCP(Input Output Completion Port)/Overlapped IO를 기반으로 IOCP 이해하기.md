## Overlapped IO를 기반으로 IOCP 이해하기

이번 챕터에서 소개하는 IOCP 서버 모델은 많은 윈도우 프로그래머의 관심사이다. 여러분도 이에 관심이 많다면 그래서 이전 내용을 건너뛰고 이리 달려온 것이라면 챕터 21 로 다시 가라

## 논의가 한참인 epoll 과 IOCP의 성능 비교

select 와 같은 전통적인 IO모델의 한계극복을 목적으로 운영체제 레벨(커널 레벨)에서 성능을 향상시킨 IO 모델이 운영체제 별로 등장하였다. 그 중 대표적인 것이 리눅스의 epoll , BSD의 kqueue 그리고 이번 챕터에서 설명하는 윈도우의 IOCP이다. 이들의 공통적인 특성은 운영체제에 의해서 기능이 지원 및 완성된다는 것이다. 그런데 여기에는 지금도 논쟁이 되는 것이 있다.

"epoll 이 빨라요? IOCP가 빨라요?"

이에 대한 논쟁은 www.yahoo.com 에서 지금도 확인할 수 있으며, 그 열기는 자칫 감정적으로 치닫는 경우도 더러 있음을 알 수 있다.
사실 서버의 응답시간과 동시접속자 수는 매우 중요한 요소이기 때문에 이에 대한 논쟁도 일리는 있다.

"epoll 로 구현한 서버에서는 동시접속자 수가 문제가 되었는데, IOCP로 바꾸니까 해결되더라고!"
"IOCP의 응답시간이 문제가 되었는데, epoll로 바꾸니깐 해결되더라고!"

그리고 하드웨어의 성능이나 할당된 대역폭이 충분한 상황에서 응답시간이나 동시접속자 수에 문제가 발생하면 필자는 다음 두 가지를 먼저 의심한다. 그리고 이 둘을 수정함으로써 대부분의 문제를 해결한다.

```
1. 비효율적인 IO의 구성 또는 비효율적인 CPU의 활용
2. 데이터베이스의 설계내용과 쿼리(Query) 구성
```

때문에 인터넷상에서 흔히 접하는 IOCP가 상대적으로 우월한 이유를 가지고 필자에게 물어오는 친구들에게는 모르겠다고 한다.

왜냐하면 다른 IO모델에 없는 장점이 IOCP에 있긴하지만, 그것이 서버의 성능을 좌우하는 절대적 기준은 아니며 모든 상황에서 그 장점이 부각되는 것도 아니기 때문이다.
오히려 이들의 성능을 좌우하는 것은 눈에 보이는 차이점이 아닌 눈에 보이지 않는 그래서 비교하기 어려운 운영체제의 내부 동작방식에 있지 않나 생각한다.
다만 필자처럼 생각하고 있는 개발자들도 주변에는 많다는 사실을 여러분에게 알리고 싶다.

# 넌-블로킹 모드의 소켓 구성하기

Overlapped IO 기반의 에코 서버를 구현하는데서 부터 이야기를 시작하고자 한다. 그런데 이에 앞서 넌-블로킹 모드로 동작하는 소켓의 생성방법부터 설명하고자 한다.
이미 챕터 17에서 넌-블로킹 모드의 소켓을 생성한 적이 있다. 이와 유사하게 윈도우에서는 다음의 함수호출을 통해서 넌-블로킹 모드로 소켓의 속성을 변경한다.

```c
SOCKET hLisnSock;
int mode = 1;

hLisnSock = WSASokcet(PF_INET , SOCK_STREAM , 0 , NULL , 0 , WSA_FLAG_OVERLAPPED);
ioctlsocket(hLisnSock , FIONBIO , &mode); // for non-blocking socket
```

위 코드에서 호출하는 ioctlsocket 함수는 소켓의 IO방식을 컨트롤하는 함수이다. 그리고 위와 같은 형태로의 함수호출이 의미하는 바는 다음과 같다.

"핸들 hLisnSock 이 참조하는 소켓의 입출력 모드 (FIONBIO)를 변수 mode 에 저장된 값의 형태로 변경한다."

즉, FIONBIO는 소켓의 입출력 모드를 변경하는 옵션이며, 이 함수의 세번째 인자로 전달된 주소 값이 변수에 0이 저장되어 있으면 블로킹모드로, 0이 아닌 값이 저장되어 있으면 넌-블로킹 모드로 소켓의 입출력 속성을 변경한다. 그리고 이렇게 속성이 넌-블로킹모드로 변경되면, 넌-블로킹 모드로 입출력되는 것 이외에 다음의 특징도 지니게 된다.

```
1. 클라이언트의 연결요청이 존재하지 않는 상태에서 accept 함수가 호출되면 INVALID_SOCKET이 곧바로 반환된다. 그리고 이어서  WSAGetLastError 함수를 호출하면 WSAEWOULDBLOCK가 반환된다.
2. accept 함수호출을 통해서 새로 생성되는 소켓 역시 넌 -블로킹 속성을 지닌다.
```

따라서 넌 블로킹 입출력 소켓을 대상으로 accept함수를 호출해서 INVALID_SOCKET이 반환되면 WSAGetLastError 함수의 호출을 통해서 invalid_socket이 반환된 이유를 확인하고, 그에 적절한 처리를 해야만 한다.

## Overlapped IO만 가지고 에코 서버 구현하기

위에서 넌-블로킹 소켓의 생성방법을 설명한 이유는 이것이 Overlapped IO 기반의 서버구현에 필요하기 때문이다.

"IOCP를 정확히 이해하기 위해서는 Overlapped IO만 가지고 서버를 구현해 봐야 한다."

```c
#include <stdio.h>
#include <stdlib.h>
#include <winsock2.h>

#define BUF_SIZE 1024

void CALLBACK ReadComRoutine(DWORD, DWORD, LPWSAOVERLAPPED, DWORD);

void CALLBACK WriteComRoutine(DWORD, DWORD, LPWSAOVERLAPPED, DWORD);

void ErrorHandling(char *message);

typedef struct {
    SOCKET hClntSock;
    char buf[BUF_SIZE];
    WSABUF wsaBuf;
} PER_IO_DATA, *LPPER_IO_DATA;

int main(int argc, const char *argv[]) {
    WSADATA wsaData;
    SOCKET hLisnSock, hRecvSock;
    SOCKADDR_IN lisnAdr, recvAdr;
    LPWSAOVERLAPPED lpOvLp;
    DWORD recvBytes;
    LPPER_IO_DATA hbInfo;
    int mode = 1, recvAdrSz, flagInfo = 0;

    if (argc != 2) {
        printf("Usage : %s <port> \n", argv[0]);
        exit(1);
    }

    if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
        ErrorHandling("WSAStartup() error!");
    }
    hLisnSock = WSASocket(PF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED);
    ioctlsocket(hLisnSock, FIONBIO, &mode);

    memset(&lisnAdr, 0, sizeof(lisnAdr));
    lisnAdr.sin_family = AF_INET;
    lisnAdr.sin_addr.S_un.S_addr = htonl(INADDR_ANY);
    lisnAdr.sin_port = htons(atoi(argv[1]));

    if (bind(hLisnSock, (SOCKADDR *) &lisnAdr, sizeof(lisnAdr)) == SOCKET_ERROR) {
        ErrorHandling("bind() error");
    }
    if (listen(hLisnSock, 5) == SOCKET_ERROR)
        ErrorHandling("listen() error");

    recvAdrSz = sizeof(recvAdr);
    while (1) {
        SleepEx(100, TRUE); // for alertable wait state
        hRecvSock = accept(hLisnSock, (SOCKADDR *) &recvAdr, &recvAdrSz);
        if (hRecvSock == INVALID_SOCKET) {
            if (WSAGetLastError() == WSAEWOULDBLOCK)
                continue;
            else
                ErrorHandling("accept() error!");
        }
        puts("Client connect..");

        lpOvLp = (LPWSAOVERLAPPED) malloc(sizeof(WSAOVERLAPPED));
        memset(lpOvLp, 0, sizeof(WSAOVERLAPPED));

        hbInfo = (LPPER_IO_DATA) malloc(sizeof(PER_IO_DATA));
        hbInfo->hClntSock = (DWORD) hRecvSock;
        (hbInfo->wsaBuf).buf = hbInfo->buf;
        (hbInfo->wsaBuf).len = BUF_SIZE;

        lpOvLp->hEvent = (HANDLE) hbInfo;
        WSARecv(hRecvSock, &(hbInfo->wsaBuf), 1, &recvBytes, &flagInfo, lpOvLp, ReadComRoutine);
    }
    closesocket(hLisnSock);
    closesocket(hRecvSock);

    return 1;
}

void CALLBACK ReadComRoutine(DWORD dwError, DWORD szRecvBytes, LPWSAOVERLAPPED lpOverlapped, DWORD flags) {
    LPPER_IO_DATA hbInfo = (LPPER_IO_DATA) (lpOverlapped->hEvent);
    SOCKET hSock = hbInfo->hClntSock;
    LPWSABUF buf_Info = &(hbInfo->wsaBuf);
    DWORD sentBytes;

    if (szRecvBytes == 0) {
        closesocket(hSock);
        free(lpOverlapped->hEvent);
        free(lpOverlapped);
        puts("Client disconnected");
    } else {
        buf_Info->len = szRecvBytes;
        WSASend(hSock, buf_Info, 1, &sentBytes, 0, lpOverlapped, WriteComRoutine);
    }
}

void CALLBACK WriteComRoutine(DWORD dwError, DWORD szSendBytes, LPWSAOVERLAPPED lpOverlapped, DWORD flags) {
    LPPER_IO_DATA hbInfo = (LPPER_IO_DATA) (lpOverlapped->hEvent);
    SOCKET hSock = hbInfo->hClntSock;
    LPWSABUF bufInfo = &(hbInfo->wsaBuf);
    DWORD recvBytes;
    int flagInfo = 0;
    WSARecv(hSock, bufInfo, 1, &recvBytes, &flagInfo, lpOverlapped, ReadComRoutine);
}

void ErrorHandling(char *message) {
    fputs(message, stderr);
    fputs('\n', stderr);
    exit(1);
}
```

위 예제의 동작원리를 정리하면 다음과 같다.

```
1. 클라이언트가 연결되면 WSARecv 함수를 호출하면서 넌 블로킹 모드로 데이터가 수신되게 하고 수신이 완료되면 ReadCOmpRoutine 함수가 호출되게 한다.

2. ReadComRoutine 함수가 호출되면 WSASend 함수를 호출하면서 넌 블로킹 모드로 데이터가 송신되게 하고, 송신이 완료되면 WriteComRoutine 함수가 호출되게 한다.

3. 그런데 이렇게 해서 호출된 WriteComRoutine 함수는 다시 WSARecv 함수를 호출하면서 넌 블로킹 모드로 데이터의 수신을 기다린다.
```

이는 매우 중요한 사실이니 다음과 같이 별도의 문장으로 간단히 정리하겠다.

"입출력 완료 시 자동으로 호출되는 Completion Routine 내부로 클라이언트 정보(소켓과 버퍼)를 전달하기 위해서 WSAOVERLAPPED 구조체의 멤버 hEvent 를 사용하였다."

이제 에코 클라이언트도 재 구현해보자

## 클라이언트의 재구현

```c
#include <stdio.h>
#include <stdlib.h>
#include <winsock2.h>

#define BUF_SIZE 1024
void ErrorHandling(char *message);

int main (int argc , char * argv[]) {

    WSADATA wsaData;
    SOCKET hSocket;
    SOCKADDR_IN servAdr;
    char message[BUF_SIZE];
    int strLen , readLen;

    if(argc != 3) {
        printf("Usage %s <IP> <port> \n" , argv[0]);
        exit(1);
    }

    if(WSAStartup(MAKEWORD(2,2) , &wsaData) != 0)
        ErrorHandling("WSAStartup() error!");

    hSocket = socket(PF_INET , SOCK_STREAM , 0);
    if(hSocket == INVALID_SOCKET)
        ErrorHandling("socket() error!");

    memset(&servAdr , 0 , sizeof(servAdr));
    servAdr.sin_family = AF_INET;
    servAdr.sin_addr.S_un.S_addr = inet_addr(argv[1]);
    servAdr.sin_port = htons(atoi(argv[2]));

    if(connect(hSocket , (SOCKADDR * ) &servAdr , sizeof(servAdr)) == SOCKET_ERROR)
        ErrorHandling("connect() error!");
    else
        puts("Connected ... ");

    while(1) {
        fputs("Input message (Q to quit)" , stdout);
        fgets(message , BUF_SIZE , stdin);
        if(!strcmp(message , "q\n") || !strcmp(message , "Q\n"))
            break;

        strLen = strlen(message);
        send(hSocket , message , strlen , 0);
        readLen = 0;
        while(1) {
            readLen += recv(hSocket , &message[readLen] , BUF_SIZE -1 , 0);
            if(readLen >= strLen)
                break;
        }
        message[strLen] = 0;
        printf("Message from server : %s" , message);
    }

    closesocket(hSocket);
    WSACleanup();
    return 0;
}

void ErrorHandling(char *message) {
    fputs(message, stderr);
    fputs('\n', stderr);
    exit(1);
}
```

## Overlapped IO 모델에서 IOCP 모델로

자 그럼 앞서 확인한 Overlapped IO 모델의 에코 서버가 지니는 단점이 무엇인지 이야기 해보자

"넌 블로킹 모드의 accept 함수와 alertable wait 상태로의 진입을 위한 SleepEx 함수가 번갈아 가며 반복 호출되는 것은 성능에 영향을 미칠 수 있다.!"

즉, 연결요청의 처리를 위한 accept 함수만 호출할 수 있는 상황이 아니기 때문에 그리고 Completion Routine 함수의 호출을 위해서 SleepEx 함수만 호출할 수 있는 상황도 아니기 때문에 accept함수는
넌 블로킹 모드로, SleepEx 함수는 타임아웃을 짧게 지정해서 돌아가며 반복 호출하였다. 그리고 이는 실제로 성능에 영향을주는 코드 구성이다.

이 문제를 해결하기 위서는 다음의 방법을 고려할 수 있다.

"accept 함수의 호출은 main 쓰레드(main 함수 내에서) 처리하도록 하고, 별도의 쓰레드를 추가로 하나 생성해서 클라이언트와 입출력을 담당하게 한다."

그리고 이것이 IOCP에서 제안하는 서버의 구현 모델이다. 즉, IOCP에서는 IO를 전담하는 쓰레드를 별도로 생성한다. 그리고 이 쓰레드가 모든 클라이언트를 대상으로 IO를 진행하게 된다.
