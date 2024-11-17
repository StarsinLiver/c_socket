## 비동기(Asynchronous) Notification IO 모델의 이해

비동기 Notification IO 모델의 구현방법에는 두가지가 존재한다. 하나는 이 책에서 설명하는 WSAEventSelect 함수를 사용하는 방법이고, 다른 하나는 WSAAsyncSelect 함수를 사용하는 방법이다. 그런데 WSAAsyncSelect 사용하기 위해서는 발생한 이벤트를 수신할 윈도우의 핸들을 지정해야 하기 때문에 (UI와 관련) 이 책에서는 언급하지 않는다.

## WSAEventSelect 함수와 Notification

IO의 상태변화를 알리는 것이 'Notification' 이다. 그런데 IO 의 상태변화는 다음과 같이 표현할 수 있다.

```
1. 소켓의 상태변화 : 소켓에 대한 IO의 상태변화
2. 소켓의 이벤트 발생 : 소켓에 대한 IO관련 이벤트의 발생
```

둘 다 IO가 필요한 또는 가능한 상황의 발생을 의미한다. 따라서 문맥에 맞게 이들 표현을 적절히 혼용하겠으니, 이에 대해서 혼동하지 않기 바란다. 그럼 먼저 WSAEventSelect 함수를 소개하겠다. 이는 임의의 소켓을 대상으로 이벤트 발생여부의 관찰을 명령할 때 사용하는 함수이다.

```c
#include <winsock2.h>

int WSAEventSelect(SOCKET s , WSAEVENT hEventObject , long lNetworkEvents) // 성공 시 0, 실패 시 SOCKET_ERROR 반환

// s : 관찰대상인 소켓의 핸들 전달
// hEventObjet : 인벤트 발생유무의 확인을 위한 Event 오브젝트의 핸들 전달
// lNetworkEvents : 감시하고자 하는 이벤트의 유형 정보 전달
```

즉, WSAEventSelect 함수는 매개변수 s에 전달된 핸들의 소켓에서 lNetworkEvents에 전달된 이벤트 중 하나가 발생하면 hEventObject에 전달된 핸들의 커널 오브젝트를 signaled 상태로 바꾸는 함수이다.
따라서 이 함수는 이렇게 불리기도 한다.

"Event 오브젝트와 소켓을 연결하는 함수"

그리고 마지막으로 중요한 사실 하나는 WSAEventSelect 함수는 이벤트의 발생유무에 상관없이 바로 반환을 하는 함수이기 때문에 함수호출 이후에 다른 작업을 진행할 수 있다는 점이다. 즉, 이 함수는 비동기 Notification 방식을 취하고 있다. 그럼 이어서 위 함수에 세번째 인자로 전달될 수 있는 이벤트의 종류에 대해 살펴보자 이들은 비트 OR 연산자를 통해서 둘 이상의 정보를 동시에 전달할 수 있다.

```
1. FD_READ : 수신할 데이터가 존재하는 가?
2. FD_WRITE : 블로킹 없이 데이터 전송이 가능한가?
3. FD_OOB : out of band 데이터가 수신되었는가?
4. FD_ACCEPT : 연결요청이 있었는가?
5. FD_CLOSE : 연결의 종료가 요청되었는가?
```

"어? select 함수는 여러 소켓을 대상으로 호출이 가능한데, WSAEventSelect 함수는 단 하나의 소켓을 대상으로만 호출이 가능하네요?"

그러고보니 수의 개념으로만 보면 WSAEventSelect 함수가 조금 밀리는 듯 보인다. 하지만 WSAEventSelect 함수를 이용하면 다수의 소켓을 대상으로 WSAEventSelect 함수를 호출할 필요를 못 느낀다. select 함수는 반환되고 나면 이벤트의 발생확인을 위해서 또 다시 모든 핸들을 대상으로 재호출해야 하지만, WSAEventSelect 함수호출을 통해서 전달된 소켓의 정보는 운영체제에 등록되고, 이렇게 등록된 소켓에 대해서는 WSAEventSelect 함수의 재호출이 불필요하기 때문이다.

## manual-reset 모드 Event 오브젝트 의 또 다른 생성방법

이전에는 CreateEvent 함수를 이용해서 Event 오브젝트를 생성하였다. CreateEvent 함수는 Event 오브젝트를 생성하되, auto-reset 모드와 manual-reset 모드 중 하나를 선택해서 생성할 수 있는 함수였다. 그러나 여기서 필요한 것은 오로지 manual-reset 모드 이면서 non-signaled 상태인 Event 오브젝트이다. 따라서 다음 함수를 이용해서 Event 오브젝트를 생성하는 것이 여러모로 편리하다

```c
#include <winsock2.h>

WINSOCK_API_LINKAGE WSAEVENT WSAAPI WSACreateEvent(void); // 성공 시 Event 오브젝트 핸들, 실패 시 WSA_INVALID_EVENT 반환
```

위의 선언에서 반환형 WSAEVENT는 다음과 같이 정의되어 있다.

```c
#define WSAEVENT HANDLE
```

이렇듯 우리가 잘 아는 커널 오브젝트의 핸들이 반환되는 것이니, 이를 다른 유형의 핸들로 구분짓지 않기 바란다. 그리고 위의 함수를 통해서 생성된 Event 오브젝트의 종료를 위한 함수는 다음과 같이 별도로 마련되어 있다.

```c
#include <winsock2.h>

WINSOCK_API_LINKAGE WINBOOL WSAAPI WSACloseEvent(WSAEVENT hEvent); // 성공 시 TRUE , 실패 시 FALSE 반환
```

## 이벤트 발생유무의 확인

WSACreateEvent 함수에 대해서도 소개했으니 WSAEventSelect 함수의 호출에는 문제가 없다 따라서 WSAEventSelect 함수호출 이후를 고민할 차례이다. 이벤트 발생유무의 확인을 위해서는 Event 오브젝트를 확인해야 한다. 이 때 사용하는 함수는 다음과 같다. 참고로 이는 매개변수가 하나 더 많다는것을 제외하면 WaitForMultipleObjects 함수와 동일하다.

```c
#include <winsock2.h>

WINSOCK_API_LINKAGE DWORD WSAAPI WSAWaitForMultipleEvents(DWORD cEvents,const WSAEVENT *lphEvents,WINBOOL fWaitAll,DWORD dwTimeout,WINBOOL fAlertable);
// 성공 시 이벤트 발생 오브젝트 관련정보, 실패 시 WSA_INVALID_EVENT 반환

// cEvents : signaled 상태로의 전이여부를 확인할 Event 오브젝트의 개수 정보 전달
// lphEvents : Event 오브젝트의 핸들을 저장하고 있는 배열의 주소 값 전달
// fWaitAll : TRUE 전달 시 모든 Event 오브젝트가 signaled 상태일 때 반환, FALSE 전달 시 하나만 signaled 상태가 되어도 반환
// dwTimeout : 1/1000 초 단위로 타임아웃 지정, WSA_INFINITE 전달시 signaled 상태가 될 때까지 반환하지 않는다.
// fAlertable : TRUE 전달 시, alertable wait 상태로의 진입
// 반환 값 : 반환된 정수 값에서 상수 값 WSA_WAIT_EVENT_0를 빼면 두번째 매개변수로 전달된 배열을 기준으로 signaled 상태가 된 Event 오브젝트의 핸들이 저장된 인덱스가 계산된다. 만약 둘 이상의 Event 오브젝트가 signald 상태로 전이 되었다면 그 중 작은 인덱스 값이 계산된다. 그리고 타임아웃이 발생하면 WAIT_TIMEOUT이 반환된다.
```

## 이벤트 종류의 구분

WSAWaitForMultipleEvents 함수를 통해서 signaled 상태로 전이된 Event 오브젝트까지 알아낼 수 있게 되었으니 , 이제 마지막으로 해당 오브젝트가 signaled 상태가 된 원인을 확인해야 한다. 그리고 이를 위해서 다음 함수를 소개한다.이 함수의 호출을 위해서는 signaled 상태의 Event 오브젝트 핸들뿐만 아니라, 이와 연결된 이벤트 발생의 주체가 되는 소켓의 핸들도 필요하다.

```c
#include <winsock2.h>

WINSOCK_API_LINKAGE int WSAAPI WSAEnumNetworkEvents(SOCKET s,WSAEVENT hEventObject,LPWSANETWORKEVENTS lpNetworkEvents);
// 성공 시 0, 실패 시 SOCKET_ERROR 반환

// s : 이벤트가 발생한 소켓의 핸들 전달
// hEventObject : 소켓과 연결된 signaled 상태인 Event 오브젝트의 핸들전달
// lpNetworkEvents : 발생한 이벤트의 유형정보와 오류정보로 채워질 WSANETWORKEVENTS 구조체 변수의 주소값 전달
```

위 함수는 manual-reset 모드의 Event 오브젝트를 non-signaled 상태로 되돌리기까지 하니, 발생한 이벤트의 유형을 확인한 다음에 별도로 ResetEvent 함수를 호출할 필요가 없다. 자! 그럼 위 함수와 관련있는 구조체 WSANETWORKEVENTS를 소개하겠다.

```c
#include <winsock2.h>

typedef struct _WSANETWORKEVENTS {
  __LONG32 lNetworkEvents;
  int iErrorCode[FD_MAX_EVENTS];
} WSANETWORKEVENTS,*LPWSANETWORKEVENTS;
```

위 구조체 맴버 INetworkEvents 에는 발생한 이벤트의 정보가 담긴다 WSAEventSelect 함수의 세번째 인자로 전달되는 상수와 마찬가지로 수신할 데이터가 존재하면 FD_READ 가 저장되고 연결요청이 있는 경우에는 FD_ACCEPT가 담긴다. 따라서 다음과 같은 방식으로 발생한 이벤트의 종류를 확인할 수 있다.

```c
WSANETWORKEVENTS netEvents;
//...
WSAEnumNetworkEvents(hSock , hEvent , &netEvents);

if(netEvents.lNetworkEvents & FD_ACCEPT) {
    // FD_ACCEPT 이벤트 발생에 대한 처리
}
if(netEvents.lNetworkEvents & FD_READ) {
    // FD_READ 이벤트 발생에 대한 처리
}
if(netEvents.lNetworkEvents & FD_CLOSE) {
    // FD_CLOSE 이벤트 발생에 대한 처리
}
```

그리고 오류발생에 대한 정보는 구조체 맴버로 선언된 배열 iErrorCode 에 담긴다. 확인할 수 있는 방법을 정리하면 다음과 같다.

```
1. 이벤트 FD_READ 관련 오류가 발생하면 iErrorCode[FD_READ_BIT] 에 0 이외의 값 저장
2. 이벤트 FD_WRITE 관련 오류가 발생하면 iErrorCode[FD_WRITE_BIT] 에 0 이외의 값 저장
```

즉, 이를 다음과 같이 일반화해서 이해하면 된다.

"이벤트 FD_XXX 관련 오류가 발생하면 iErrorCode[FD_XXX_BIT]에 0 이외의 값 저장"

따라서 다음의 형태로 오류검사를 진행하면 된다.

```c
WSANETWORKEVENTS netEvents;
//...
WSAEnumNetworkEvents(hSock , hEvent , &netEvents);

if(netEvents.iErrorCode[FD_READ_BIT] != 0) {
    // FD_READ 이벤트 관련 오류 발생
}
```

그럼 이로써 비동기 Asunchronous Notification IO 모델에 대한 설명을 마치고 지금까지 이해한 내용을 모아서 하나의 예제를 작성해 보겠따.

```c
#include <stdio.h>
#include <winsock2.h>
#include <string.h>

#define BUF_SIZE 100

void CompressSockets(SOCKET hSocketArr[], int idx, int total);

void CompressEvents(WSAEVENT hEventArr[], int idx, int total);

void ErrorHandling(char *msg);

int main(int argc, char *argv[]) {
    WSADATA wsaData;
    SOCKET hServSock, hClntSock;
    SOCKADDR_IN servAdr, clntAdr;

    SOCKET hSockArr[WSA_MAXIMUM_WAIT_EVENTS];
    WSAEVENT hEventArr[WSA_MAXIMUM_WAIT_EVENTS];
    WSAEVENT newEvent;
    WSANETWORKEVENTS netEvents;

    int numOfClntSock = 0;
    int strLen, i;
    int posInfo, startIdx;
    int clntAdrLen;
    char msg[BUF_SIZE];

    if (argc != 2) {
        printf("Usage : %s <port>\n", argv[0]);
        exit(1);
    }

    if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0) {
        ErrorHandling("WSAStartup() error!");
    }

    hServSock = socket(PF_INET, SOCK_STREAM, 0);
    memset(&servAdr, 0, sizeof(servAdr));
    servAdr.sin_family = AF_INET;
    servAdr.sin_addr.S_un.S_addr = htonl(INADDR_ANY);
    servAdr.sin_port = htons(atoi(argv[1]));

    if (bind(hServSock, (SOCKADDR *) &servAdr, sizeof(servAdr)) == SOCKET_ERROR) {
        ErrorHandling("bind() error!");
    }

    if (listen(hServSock, 5) == SOCKET_ERROR)
        ErrorHandling("listen() error");


    newEvent = WSACreateEvent();
    if (WSAEventSelect(hServSock, newEvent, FD_ACCEPT) == SOCKET_ERROR) {
        ErrorHandling("WSAEventSelect() error!");
    }

    hSockArr[numOfClntSock] = hServSock;
    hEventArr[numOfClntSock] = newEvent;
    numOfClntSock++;

    while (1) {
        posInfo = WSAWaitForMultipleEvents(numOfClntSock, hEventArr, FALSE, WSA_INFINITE, FALSE);
        startIdx = posInfo - WSA_WAIT_EVENT_0;

        for (i = startIdx; i < numOfClntSock; i++) {
            int sigEventIdx = WSAWaitForMultipleEvents(1, &hEventArr[i], TRUE, 0, FALSE);
            if (sigEventIdx == WSA_WAIT_FAILED || sigEventIdx == WSA_WAIT_TIMEOUT) {
                continue;
            } else {
                sigEventIdx = i;
                WSAEnumNetworkEvents(hSockArr[sigEventIdx], hEventArr[sigEventIdx], &netEvents);

                // 연결 요청시
                if (netEvents.lNetworkEvents & FD_ACCEPT) {
                    if (netEvents.iErrorCode[FD_ACCEPT_BIT] != 0) {
                        puts("Accept Error");
                        break;
                    }

                    clntAdrLen = sizeof(clntAdr);
                    hClntSock = accept(hSockArr[sigEventIdx], (SOCKADDR *) &clntAdr, &clntAdrLen);
                    newEvent = WSACreateEvent();
                    WSAEventSelect(hClntSock, newEvent, FD_READ | FD_CLOSE);

                    hEventArr[numOfClntSock] = newEvent;
                    hSockArr[numOfClntSock] = hClntSock;
                    numOfClntSock++;
                    puts("conneted new Client");
                }

                // 데이터 수신시
                if (netEvents.lNetworkEvents & FD_READ) {
                    if (netEvents.iErrorCode[FD_READ_BIT] != 0) {
                        puts("Rad Error");
                        break;
                    }

                    strLen = recv(hSockArr[sigEventIdx], msg, sizeof(msg), 0);
                    send(hSockArr[sigEventIdx], msg, strLen, 0);
                }

                // 종료 요청시
                if (netEvents.lNetworkEvents & FD_CLOSE) {
                    if (netEvents.iErrorCode[FD_CLOSE_BIT] != 0) {
                        puts("Close Error");
                        break;
                    }

                    WSACloseEvent(hEventArr[sigEventIdx]);
                    closesocket(hSockArr[sigEventIdx]);

                    numOfClntSock--;
                    CompressSockets(hSockArr, sigEventIdx, numOfClntSock);
                    CompressEvents(hEventArr, sigEventIdx, numOfClntSock);
                }
            }
        }
    }

    WSACleanup();
    return 0;
}


void CompressSockets(SOCKET hSocketArr[], int idx, int total) {
    int i;
    for(i = idx , i < total , i++) {
        hSocketArr = hSocketArr[i + 1];
    }
}

void CompressEvents(WSAEVENT hEventArr[], int idx, int total) {
    int i;
    for(i = idx; i < total; i++) {
        hEventArr[i] = hEventArr[i + 1];
    }
}

void ErrorHandling(char *msg) {
    fputs(msg , stderr);
    fputc('\n' , stderr);
    exit(1);
}
```

윈도우 에코 클라이언트를 아무거나 실행해보면 된다.
