## IOCP의 단계적 구현 - Completion Port의 생성

IOCP에서는 완료된 IO의 정보가 Completion Port 오브젝트 (이하 CP오브젝트)라는 커널 오브젝트에 등록된다. 그런데 그냥 등록되는 것이 아니기 때문에 다음과 같은 요청의 과정이 선행되어야 한다.

"이 소켓을 기반으로 진행되는 IO의 완료 상황은 저 CP오브젝트에 등록해 주세요"

이를 가리켜 "소켓과 CP오브젝트와의 연결 요청" 이라 한다. 때문에 IOCP 모델의 서버 구현을 위해서는 다음 두 가지 일을 진행해야 한다.

```
1. Completion Port 오브젝트의 생성
2. Completion Port 오브젝트와 소켓의 연결
```

이때 소켓은 반드시 Overlapped 속성이 부여된 소켓이어야 하며, 위의 두 가지 일은 다음 하나의 함수를 통해서 이뤄진다. 그러나 일단 CP오브젝트의 생성 관점에서 다음 함수를 보자

```c
#include <windows.h>

WINBASEAPI HANDLE WINAPI CreateIoCompletionPort (HANDLE FileHandle, HANDLE ExistingCompletionPort, ULONG_PTR CompletionKey, DWORD NumberOfConcurrentThreads); // 성공 시 CP 오브젝트의 핸들 , 실패시 NULL 반환

// FileHandle : CP 오브젝트 생성시에는 INVALID_HANDLE_VALUE를 전달.
// ExistingCompletionPort : CP 오브젝트 생성시에는 NULL 전달
// CompletionKey : CP 오브젝트 생성시에는 0 전달
// NumberOfConcurrentThreads : CP 오브젝트에 할당되어 완료된 IO를 처리할 쓰레드의 수를 전달, 예를 들어 2가 전달되면 CP오브젝트에 할당되어 동시 실행 가능한 쓰레드의 수는 최대 2개로 제한된다. 만약 0 이 전달되면 시스템의 CPU 개수가 동시 실행 가능한 쓰레드의 최대수로 지정된다.
```

위 함수를 CP 오브젝트의 생성을 목적으로 호출할 때에는 마지막 매개변수만이 의미를 갖는다.

즉 다음과 같이 할 수 있다.

```c
HANDLE hCpObject = CreateIoCompletionPort(INVALID_HANDLE_VALUE , NULL , 0 , 2);
```

## Completion Port 오브젝트와 소켓의 연결

CP 오브젝트가 생성되었다면, 이제 이를 소켓과 연결시켜야 한다. 그래야 완료된 소켓의 IO정보가 CP 오브젝트에 등록된다. 그럼 이번에는 이를 목적으로 CreaetIoCompletionPort함수를 다시 한번 소개하겠다.

```c
#include <windows.h>

WINBASEAPI HANDLE WINAPI CreateIoCompletionPort (HANDLE FileHandle, HANDLE ExistingCompletionPort, ULONG_PTR CompletionKey, DWORD NumberOfConcurrentThreads);

// FileHandle : CP 오브젝트에 연결할 소켓의 핸들 전달.
// ExistingCompletionPort : 소켓과 연결할 CP 오브젝트의 핸들전달
// CompletionKey : 완료된 IO 관련 정보의 전달을 위한 매개변수
// NumberOfConcurrentThreads : 어떠한 값을 전달하건, 이 함수의 두번째 매개변수가 NULL 이 아니면 그냥 무시됨
```

즉, 매개변수 FileHandle 에 전달된 핸들의 소켓을 매개변수 ExistingCompletionPort 에 전달된 핸들의 CP오브젝트에 연결시키는 것이 위 함수의 두번째 기능이다. 그리고 호출의 형태는 다음과 같다.

```c
HANDLE hCpObject;
SOCKET hSocket;

CreateIoCompletionPort((HANDLE) hSocket , hCpObject , (DWORD) ioInfo , 0);
```

이렇게 CreateIoCompletionPort 함수가 호출된 이후부터는 hSock을 대상으로 진행된 IO가 완료되면 이에 대한 정보가 핸들 hCpObject 에 해당하는 CP오브젝트에 등록된다.

## Completion Port의 완료된 IO 확인과 쓰레드의 IO 처리

CP 오브젝트의 생성 및 소켓과의 연결방법도 알았으니, 이제 CP에 등록되는 완료된 IO 의 확인 방법을 살펴볼 차례이다. 이에 사용되는 함수는 다음과 같다.

```c
#include <windows.h>

WINBASEAPI WINBOOL WINAPI GetQueuedCompletionStatus (HANDLE CompletionPort, LPDWORD lpNumberOfBytesTransferred, PULONG_PTR lpCompletionKey, LPOVERLAPPED *lpOverlapped, DWORD dwMilliseconds);
// 성공 시 TRUE , 실패 시 FALSE 반환

// CompletionPort : 완료된 IO 정보가 등록되어 있는 CP 오브젝트 전달.
// lpNumberOfBytes : 입출력 과정에서 송수신된 데이터의 크기 정보를 저장할 변수의 주소 값 전달
// lpCompletionKey : CreateIoCompletionPort 함수의 세번째 인자로 전달된 값의 저장을 위한 변수의 주소 값 전달
// lpOverlapped : WSASend , WSARecv 함수호출 시 전달하는 OVERLAPPED구조체 변수의 주소 값이 저장될, 변수의 주소 값 전달
// dwMilliseconds : 타임아웃 정보 전달, 여기서 지정한 시간이 완료되면 FALSE 를 반환하면서 함수를 빠져나가며 INFINITE를 전달하면 완료된 IO가 CP 오브젝트에 등록될 때까지 블로킹 상태에 있게된다.
```

지금껏 IOCP와 관련해서 추가로 설명된 함수는 두 개에 지나지 않는다. 그럼에도 불구하고 조금 혼란스러울 수 있다. 특히 위 함수의 세 번째 그리고 네 번째 매개변수의 경우 더 그렇게 생각할 수 있다.
이 두 개의 매개변수를 통해서 얻게 되는 정보가 무엇인지 정리해보겠다.

"GetQueuedCompletionStatus 함수의 세번째 인자를 통해서 얻게되는 것은 소켓과 CP오브젝트의 연결을 목적으로 CreateIoCompletionPort 함수가 호출될 때 전달되는 세번째 인자 값이다."

"GetQueuedCompletionStatus 함수의 네 번째 인자를 통해서 얻게되는 것은 WSASend , WSARecv 함수호출 시 전달되는 WSAOVERLAPPED 구조체 변수의 주소값이다"

실제로 이 둘이 어떻게 활용되는지 예제를 통해서 여러분이 별도로 이해해야 하니 일단 이 정도만 정리해두자.

"그럼 IO에 어떻게 쓰레드를 할당해야 하나요?"

앞서 IOCP 에서는 IO를 전담하는 쓰레드를 별도로 생성하고 이 쓰레드가 모든 클라이언트를 대상으로 IO를 진행하게 된다고 하였다. 그리고 이 내용에 힘을 실어주듯이 CreateIoCompletionPort 함수에는 생성되는 CP 오브젝트에 할당할 최대 쓰레드의 수를 지정하는 매개변수도 존재한다.

"혹시 쓰레드가 자동으로 생성되어서 IO를 처리하는가?"

아니다! 입출력 할수인 WSASend , WSARecv 함수를 호출하는 쓰레드는 우리가 직접 생성해야 한다. 다만 이 쓰레드가 입출력의 완료를 위해서 GetQueuedCompletionStatus 함수를 호출할 뿐이다. 그리고 GetQueuedCompletionStatus 함수는 어떤한 쓰레드라도 호출 가능하지만, 실제 IO의 완료에 대한 응답을 받는 쓰레드의 수는 CreateIoCompletionPort 함수호출시 지정한 최대 쓰레드의 수를 넘지 않는다.

## IOCP 기반의 에코 서버의 구현

```c
#include <stdio.h>
#include <stdlib.h>
#include <process.h>
#include <winsock2.h>
#include <windows.h>

#define BUF_SIZE 1024
#define READ 3
#define WRITE 5

typedef struct {
    SOCKET hClntSock;
    SOCKADDR_IN clntAdr;
} PER_HANDLE_DATA, *LPPER_HANDLE_DATA;

typedef struct {
    OVERLAPPED overlapped;
    WSABUF wsaBuf;
    char buffer[BUF_SIZE];
    int rwMode; // READ or WRITE
} PER_IO_DATA, *LPPER_IO_DATA;

DWORD WINAPI EchoThreadMain(LPVOID CompletionPortIO);

void ErrorHandling(char *message);

int main(int argc, char *argv[]) {
    WSADATA wsaData;
    HANDLE hComPort;
    SYSTEM_INFO sysInfo;
    LPPER_IO_DATA ioInfo;
    LPPER_HANDLE_DATA handleInfo;

    SOCKET hServSock;
    SOCKADDR_IN servAdr;
    int recvBytes, i, flags = 0;

    if (WSAStartup(MAKEWORD(2, 2), &wsaData) != 0)
        ErrorHandling("WSAStartup() error!");

    hComPort = CreateIoCompletionPort(INVALID_HANDLE_VALUE, NULL, 0, 0);
    GetSystemInfo(&sysInfo);

    for (i = 0; i < sysInfo.dwNumberOfProcessors; i++) {
        _beginthreadex(NULL, 0, EchoThreadMain, (LPVOID) hComPort, 0, NULL);
    }

    hServSock = WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED);
    memset(&servAdr, 0, sizeof(servAdr));
    servAdr.sin_family = AF_INET;
    servAdr.sin_addr.S_un.S_addr = htonl(INADDR_ANY);
    servAdr.sin_port = htons(atoi(argv[1]));

    bind(hServSock, (SOCKADDR *) &servAdr, sizeof(servAdr));
    listen(hServSock, 5);

    while (1) {
        SOCKET hClntSock;
        SOCKADDR_IN clntAdr;
        int addrLen = sizeof(clntAdr);
        hClntSock = accept(hServSock, (SOCKADDR *) &clntAdr, &addrLen);
        handleInfo = (LPPER_HANDLE_DATA) malloc(sizeof(PER_HANDLE_DATA));
        handleInfo->hClntSock = hClntSock;
        handleInfo->clntAdr = clntAdr;
        memcpy(&(handleInfo->clntAdr), &clntAdr, addrLen);

        CreateIoCompletionPort((HANDLE) hClntSock, hComPort, (DWORD) handleInfo, 0);

        ioInfo = (LPPER_IO_DATA) malloc(sizeof(PER_IO_DATA));
        memset(&(ioInfo->overlapped), 0, sizeof(PER_IO_DATA));
        ioInfo->wsaBuf.len = BUF_SIZE;
        ioInfo->wsaBuf.buf = ioInfo->buffer;
        ioInfo -> rwMode = READ;
        WSARecv(handleInfo ->hClntSock , &(ioInfo ->wsaBuf) , 1 , &recvBytes , &flags , &(ioInfo -> overlapped) ,  NULL);
    }

    return 0;
}

DWORD WINAPI EchoThreadMain(LPVOID pComPort) {
    HANDLE hComPort = (HANDLE) pComPort;
    SOCKET sock;
    DWORD bytesTrans;
    LPPER_HANDLE_DATA handleInfo;
    LPPER_IO_DATA ioInfo;
    DWORD flags = 0;

    while(1) {
        GetQueuedCompletionStatus(hComPort , &bytesTrans , (LPWORD) &handleInfo , (LPOVERLAPPED*) &ioInfo , INFINITE);
        sock = handleInfo -> hClntSock;

        if(ioInfo -> rwMode == READ) {
            puts("message Receive!");

            if(bytesTrans == 0) { // EOF 전송시
                closesocket(sock);
                free(handleInfo); free(ioInfo);
                continue;
            }

            memset(&(ioInfo -> overlapped) , 0 , sizeof(OVERLAPPED));
            ioInfo->wsaBuf.len = bytesTrans;
            ioInfo ->rwMode = WRITE;
            WSASend(sock , &(ioInfo -> wsaBuf) , 1 , NULL,  0 , &(ioInfo ->overlapped) , NULL);

            ioInfo = (LPPER_IO_DATA) malloc(sizeof(PER_IO_DATA));
            memset(&(ioInfo -> overlapped) , 0 , sizeof(OVERLAPPED));
            ioInfo ->wsaBuf.len = BUF_SIZE;
            ioInfo -> wsaBuf.buf = ioInfo -> buffer;
            ioInfo -> rwMode = READ;
            WSARecv(sock , &(ioInfo -> wsaBuf) , 1 , NULL , &flags , &(ioInfo -> overlapped) , NULL);
        }
        else {
            puts("message send!");
            free(ioInfo);
        }
    }

}

void ErrorHandling(char *message) {
    fputs(message, stderr);
    fputs("\n", stderr);
    exit(1);
}
```

```
gcc .\main.c -o main -lws2_32
```

## IOCP 가 성능이 좀 더 나오는 이유

그냥 막연하게 "IOCP이니까 성능이 좋을 것이다." 라는 생각은 문제가 있다. 지금까지 리눅스와 윈도우를 바탕으로 다양한 서버의 모델을 제시하였으니 우리가 판단해보자.

코드 레벨에서 select 모델과 비교해보면 다음 두 가지 특성을 발견할 수 있다.

```
1. 넌 블로킹 방식으로 IO가 진행되기 때문에 IO 작업으로 인한 시간의 지연이 발생하지 않는다.
2. IO가 완료된 핸들을 찾기 위해서 반복문을 구성할 필요가 없다.
3. IO의 진행대상인 소켓의 핸들을 배열에 저장해 놓고 관리할 필요가 없다.
4. IO의 처리를 위한 쓰레드의 수를 조절할 수 있다. 따라서 실험적 겨로가를 토대로 적절한 쓰레드의 수를 지정할 수 있따.
```

이 정도만 보더라도 IOCP는 좋은 성능을 낼 수 있는 구조임에 들림없다. 그런데 IOCP는 윈도우 운영체제에 의해서 지원되는 기능이기 때문에 추가적인 성능향상의 요인을 운영체제가 별도로 제공하고 있다.
그래서 IOCP는 리눅스의 epoll과 함께 너무나도 휼륭한 서버 모델이라고 하지 않았는가!
