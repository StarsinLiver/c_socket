## Overlapped IO에서의 입출력 완료의 확인

overlapped IO에서 입출력의 완료 및 결과를 확인하는 방법에는 두가지가 있다. 그 두 가지는 다음과 같다.

```
1. WSASend , WSARecv 함수의 여섯번째 매개변수 활용 방법, Event 오브젝트 기반
2. WSASend , WSARecv 함수의 일곱번째 매개변수 활용 방법 , Completion Routine 기반
```

이 둘을 이해해야 윈도우에서 말하는 Overlapped IO 를 이해하는 셈이 된다. 그럼 먼저 여섯번째 여섯번째 매개변수를 활용하는 방법부터 소개를 하겠다.

## Event 오브젝트 사용하기

WSASend , WSARecv 함수의 여섯 번째 인자로 전달되는 WSAOVERLAPPED 구조체 변수에 대해서는 앞서 설명하였으니, 예제를 보이는 것이 우선일 듯 하다. 그럼 예제를 통해서 다음 두 가지 사실을 확인하기 바란다.

```
1. IO가 완료되면 WSAOVERLAPPED 구조체 변수가 참조하는 Event 오브젝트가 signaled 상태가 된다.
2. IO의 완료 및 결과를 확인하려면 WSAGetOverlappedResult 함수를 사용한다.
```

참고로 아래의 예제는 지금까지 설명한 내용을 정리할 수 있는 수준의 예제일 뿐이다. 그러니 이 예제를 바탕으로 Overlapped IO의 장점을 부각시키는 예제를 별도로 작성하는 기회를 가져보기 바란다.

```c
#include <stdio.h>
#include <winsock2.h>
#include <stdlib.h>

void ErrorHandling(char *msg);

int main(int argc, char *argv[]) {
    WSADATA wsaData;
    SOCKET hSocket;
    SOCKADDR_IN sendAdr;

    WSABUF dataBuf;
    char msg[] = "Network is Computer";
    int sendBytes = 0;

    WSAEVENT evObj;
    WSAOVERLAPPED overlapped;

    if (argc != 3) {
        printf("Usage : %s <IP> <PORT> \n", argv[0]);
        exit(1);
    }

    if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
        ErrorHandling("WSAStartup() error");
    }

    hSocket = WSASocket(PF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED);

    memset(&sendAdr, 0, sizeof(sendAdr));
    sendAdr.sin_family = AF_INET;
    sendAdr.sin_addr.S_un.S_addr = inet_addr(argv[1]);
    sendAdr.sin_port = htons(atoi(argv[2]));

    if (connect(hSocket, (SOCKADDR *) &sendAdr, sizeof(sendAdr)) == SOCKET_ERROR)
        ErrorHandling("connect() error!");

    evObj = WSACreateEvent();
    memset(&overlapped, 0, sizeof(overlapped));
    overlapped.hEvent = evObj;
    dataBuf.len = strlen(msg) + 1;
    dataBuf.buf = msg;

    if (WSASend(hSocket, &dataBuf, 1, &sendBytes, 0, &overlapped, NULL) == SOCKET_ERROR) {
        if (WSAGetLastError() == WSA_IO_PENDING) {
            puts("Background data send");
            WSAWaitForMultipleEvents(1, &evObj, TRUE, WSA_INFINITE, FALSE);
            WSAGetOverlappedResult(hSocket, &overlapped, &sendBytes, FALSE, NULL);

        }
        else {
            ErrorHandling("WSASend() error!");
        }
    }

    printf("Send data size : %d \n" , sendBytes);
    WSACloseEvent(evObj);
    closesocket(hSocket);
    WSACleanup();

    return 0;
}

void ErrorHandling(char *msg) {
    fputs(msg, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

참고로 위 예제에서 호출한 WSAGetLastError 함수는 다음과 같이 정의되어 있어서 소켓관련 함수가 호출된 이후에 발생하는 오류의 원인 정보를 반환한다.

```c
#include <winsock2.h>

WINSOCK_API_LINKAGE int WSAAPI WSAGetLastError(void);
```

위 예제에서도 이 함수가 반환한 값 WSA_IO_PENDING을 통해서 WSASend 함수의 호출결과가 오류상황이 아닌 완료되지 않은 상황임을 확인할 수 있다.
그럼 이어서 위 예제와 함께 동작하는 Receiver 를 보이겠다. 참고로 이 예제 역시 기본적인 구성은 앞서 보인 Sender 와 유사하다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <winsock2.h>

#define BUF_SIZE 1024

void ErrorHandling(char *message);

int main(int argc, char *argv[]) {
    WSADATA wsaData;
    SOCKET hLisnSock, hRecvSock;
    SOCKADDR_IN lisnAdr, recvAdr;
    int recvAdrSz;

    WSABUF dataBuf;
    WSAEVENT evObj;
    WSAOVERLAPPED overlapped;

    char buf[BUF_SIZE];
    int recvBytes = 0, flags = 0;

    if (argc != 2) {
        printf("Usage : %s <port> \n", argv[0]);
        exit(1);
    }

    if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0)
        ErrorHandling("WSAStartup() error!");

    hLisnSock = WSASocket(PF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED);
    memset(&lisnAdr , 0 , sizeof(lisnAdr));
    lisnAdr.sin_family = AF_INET;
    lisnAdr.sin_addr.S_un.S_addr = htonl(INADDR_ANY);
    lisnAdr.sin_port = htons(atoi(argv[1]));

    if(bind(hLisnSock , (SOCKADDR*) &lisnAdr , sizeof(lisnAdr)) == SOCKET_ERROR) {
        ErrorHandling("bind() error!");
    }
    if(listen(hLisnSock , 5) == SOCKET_ERROR) {
        ErrorHandling("listen() error!");
    }

    recvAdrSz = sizeof(recvAdr);
    hRecvSock = accept(hLisnSock , (SOCKADDR*) &recvAdr , &recvAdrSz);

    evObj = WSACreateEvent();
    memset(&overlapped , 0 , sizeof(overlapped));
    overlapped.hEvent = evObj;
    dataBuf.len = BUF_SIZE;
    dataBuf.buf = buf;

    if(WSARecv(hRecvSock , &dataBuf , 1 , &recvBytes , &flags , &overlapped , NULL) == SOCKET_ERROR) {
        if(WSAGetLastError() == WSA_IO_PENDING) {
            puts("Background data receive");
            WSAWaitForMultipleEvents(1 , &evObj , TRUE , WSA_INFINITE , FALSE);
            WSAGetOverlappedResult(hRecvSock , &overlapped , &recvBytes , FALSE , NULL);
        } else {
            ErrorHandling("WSARecv() error!");
        }

    }

    printf("Received message : %s \n" , buf);
    WSACloseEvent(evObj);
    closesocket(hRecvSock);
    closesocket(hLisnSock);
    WSACleanup();
    return 0;
}

void ErrorHandling(char *message) {
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

## Completion Routine 사용하기

앞에서는 IO의 완료를 Event 오브젝트를 이용해서 확인하였는데, 이번에는 WSASend , WSARecv 함수의 마지막 전달인자를 통해서 등록되는, Completion Routine 이라 불리는 함수를 통해서 확인하는 방법을 소개하겠다. 이러한 CR의 등록은 다음의 의미를 갖는다.

"Pending된 IO가 완료되면, 이 함수를 호출해 달라"

이렇듯 IO가 완료되었을 때, 자동으로 호출될 함수를 등록하는 형태로 IO완료 이후의 작업을 처리하는 방식이 Completion Routine 을 활용하는 방식이다. 그런데 매우 중요한 작업을 진행중인 상황에서 갑자기 Completion Routine 이 호출되면 프로그램의 흐름을 망칠 수 있다. 따라서 운영체제는 다음과 같이 이야기 한다.

"IO를 요청한 쓰레드가 alertable wait 상태에 놓여있을때만 Completion Routine 을 호출할게!"

'alertable wait 상태' 라는 것은 운영체제가 전달하는 메시지의 수신을 대기하는 쓰레드의 상태를 뜻하며, 다음 함수가 호출된 상황에서 쓰레드는 alertable wait 상태가 된다.

```
1. WaitForSingleObjetEx
2. WaitForMultipleObjectsEx
3. WSAWaitForMultipleEvents
4. SleepEx
```

이 중에서 첫 번째 두 번째 그리고 네 번째 함수는 여러분이 잘 아는 WaitForSingleObject , WaitForMultipleObjects 그리고 Sleep 함수와 동일한 기능을 제공한다. 단, 위 함수들은 이들보다 매개변수가 마지막에 하나 더 추가되어 있는데 이 매개변수에 TRUE를 전달하면 해당 쓰레드는 alertable wait 상태가 된다. 그리고 WSA로 시작하는 이름의 함수는 앞서 챕터 21 에서 소개한 바 있는데, 이 함수 역시 마지막 매개변수로 TRUE 가 전달되면 해당 쓰레드는 alertable wait 상태가 된다.

따라서 IO 를 진행시킨 다음에 급한 다른 볼일들을 처리하고 나서, IO가 완료되었는지 확인하고 싶을때 위의 함수들 중 하나를 호출하면 된다. 그러면 운영체제는 쓰레드가 alertable wait 상태에 진입한것을 인식하고 완료된 IO가 있다면 이에 해당하는 ompletion Routine 을 호출해준다.

물론 Completion Routine 이 실행되면 위 함수들을 모두 WAIT_IO_COMPLETION을 반환하면서 함수를 빠져나온다. 그리고는 그 다음부터 실행을 이어나간다.

이로써 Completion Routine 과 관련해서도 필요한 이론적인 설명을 모두 마쳤따. 따라서 앞서 구현했던 OverlappedRecv_win.c 을 Completion Routine 기반으로 변경하면서 설명을 마치고자 한다.

```c
#include <stdio.h>
#include <stdlib.h>
#include <winsock2.h>

#define BUF_SIZE 1024
void CALLBACK CompRoutine(DWORD , DWORD , LPWSAOVERLAPPED , DWORD);
void ErrorHandling(char *message);

WSABUF dataBuf;
char buf[BUF_SIZE];
int recvBytes = 0;

int main(int argc, char *argv[]) {
    WSADATA wsaData;
    SOCKET hLisnSock, hRecvSock;
    SOCKADDR_IN lisnAdr, recvAdr;

    WSAEVENT evObj;
    WSAOVERLAPPED overlapped;

    int idx , recvAdrSz , flags = 0;

    if (argc != 2) {
        printf("Usage : %s <port> \n", argv[0]);
        exit(1);
    }

    if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0)
        ErrorHandling("WSAStartup() error!");

    hLisnSock = WSASocket(PF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED);
    memset(&lisnAdr , 0 , sizeof(lisnAdr));
    lisnAdr.sin_family = AF_INET;
    lisnAdr.sin_addr.S_un.S_addr = htonl(INADDR_ANY);
    lisnAdr.sin_port = htons(atoi(argv[1]));

    if(bind(hLisnSock , (SOCKADDR*) &lisnAdr , sizeof(lisnAdr)) == SOCKET_ERROR) {
        ErrorHandling("bind() error!");
    }
    if(listen(hLisnSock , 5) == SOCKET_ERROR) {
        ErrorHandling("listen() error!");
    }

    recvAdrSz = sizeof(recvAdr);
    hRecvSock = accept(hLisnSock , (SOCKADDR*) &recvAdr , &recvAdrSz);
    if(hRecvSock == INVALID_SOCKET)
        ErrorHandling("accept error!");

    memset(&overlapped , 0 , sizeof(overlapped));
    dataBuf.len = BUF_SIZE;
    dataBuf.buf = buf;
    evObj = WSACreateEvent();

    if(WSARecv(hRecvSock , &dataBuf , 1 , &recvBytes , &flags , &overlapped , CompRoutine) == SOCKET_ERROR) {
        if(WSAGetLastError() == WSA_IO_PENDING) {
            puts("Beackgorund data Receive");
        }

    }
    idx = WSAWaitForMultipleEvents(1 , &evObj , FALSE , WSA_INFINITE , TRUE);
    if(idx == WAIT_IO_COMPLETION) {
        puts("Overlapped I/O Completed");
    } else {
        ErrorHandling("WSARecv() error!");
    }

    WSACloseEvent(evObj);
    closesocket(hRecvSock);
    closesocket(hLisnSock);
    WSACleanup();
    return 0;
}

void CALLBACK CompRoutine(DWORD dwError , DWORD szRecvBytes , LPWSAOVERLAPPED lpOverlapped , DWORD flags) {
    if(dwError != 0) {
        ErrorHandling("CompRoutine error!");
    }
    else {
        recvBytes = szRecvBytes;
        printf("Received message : %s \n" , buf);
    }
}

void ErrorHandling(char *message) {
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}
```

다음은 WSARecv 함수의 마지막 인자로 전달된 Completion Routine 의 원형이다.

```c
void CALLBACK CompRoutine(DWORD dwError , DWORD szRecvBytes , LPWSAOVERLAPPED lpOverlapped , DWORD flags)
```

이 중에서 첫 번째 매개변수로는 오류정보가 두 번째 매개변수로는 완료된 입출력 데이터의 크기정보가 전달된다. 그리고 세 번째 매겹ㄴ수로는 WSASend , WSARecv 함수의 매개변수 lpOverlapped 로 전달된 값이 , 마지막으로 dwFlags 에는 입출력 함수호출 시 전달된 특성정보 또는 0에 전달된다. 그리고 반환형 void 옆에 삽입된 키워드 CALLBACK 은 쓰레드의 main 함수에 선언되는 키워드인 WINAPI 와 마찬가지로 함수의 호출규약을 선언해 놓은 것이니, Completion Routine 을 정의하는 경우에는 반드시 삽입하기 바란다.

IOCP의 이해적으로 진행되었다고 해도 과언이 아니다.
