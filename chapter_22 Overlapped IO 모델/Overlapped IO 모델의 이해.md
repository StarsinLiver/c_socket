## Overlapped IO 모델의 이해

챕터 21 에서 비동기로 처리되었던 것은 IO 가 아닌 Notification(알림) 이었다. 그러나 여기서는 IO를 비동기로 처리하는 방법에 대해서 설명한다. 이 둘의 차이점과 각각의 장점을 명확히 구분할 수 있어야 IOCP라는 것을 쉽게 공부할 수 있다.

## IO(입출력)의 중첩이란?

사실 IO 의 중첩이라는 것이 여러분에게 생소한 개념은 아니다. 여러분은 이미 비동기 IO가 무엇인지 알고 있지 않은가? 비동기 IO 모델을 설명하였는데 그런데 바로 이 비동기 IO가 사실상 Overlapped(중첩된) IO나 다름이 없다. 그럼 필자가 Overlapped IO를 설명할 테니 우리가 직접 판단해보자

![alt text](/image/38.png)

위 그림에서 보이듯이 하나의 쓰레드 내에서 동시에 둘 이상의 영역으로 데이터를 전송 또는 수신함으로 인해서 입출력이 중첩되는 상황을 가리켜 'IO의 중첩'이라한다. 그리고 이러한 일이 가능하려면 호출된 입출력 함수가 바로 반환해야 한다. 그래야 두 번째 , 세번째 데이터 전송을 시도할 수 있끼 때문이다. 결과적으로 위의 모델로 데이터를 송수신하는데 있어서 핵심이 되는 사항은 '비동기 IO'이다. 그리고 비동기 IO가 가능하려면 호출되는 입출력 함수는 넌-블로킹 모드로 동작해야 한다.

그럼 우리가 판단할 차례이다. '비동기 IO' 와 'Overlapped IO'에 차이가 있는가? 차이가 있다고 말해도 좋고 차이가 없다고 말해도 좋다 중요한것은 이 둘의 관계를 이해하는 것이다. 비동기 방식으로 IO를 진행하는 경우 이번 chapter 에서 소개하는 방식을 사용하지 않더라도 위 그림의 형태로 입출력을 구성할 수 있기 때문에 필자는 이 둘을 굳이 구분할 필요가 없다는 것이다.

## 어번 챕터에서 말하는 Overlapped IO의 포커스는 IO 에 있지 않습니다.

비동기 IO 와 Overlapped IO를 비교 이해하였으니 이번 챕터에서 설명하려는 것을 이론적으로 나마 다 이해한 것 처럼 생각할 수 있다. 하지만 우리는 Overlapped IO 를 시작조차 하지 않았다.

윈도우에서 말하는 overlapped IO의 포커스는 IO가 아닌 IO 완료된 상황의 확인방법에 있기 때문이다. 입력을 하건 출력을 하건 이들이 넌-블로킹 모드로 진행된다면 이후에 완료결과를 별도로 확인해야 한다. 그런데 이의 확인방법에 대해 아는 바가 없지 않은가? 이의 확인을 위해서는 별도의 과정을 거쳐야 하는데 이를 바로 이번 챕터에서 이야기하려는 것이다. 즉, 윈도우에서 말하는 Overlapped IO 는 그림에서 보이는 방식으로의 입출력만을 뜻하는 것이 아니고, 입출력의 완료를 확인하는 방법까지 포함한 것이다.

## Overlapped IO 소켓의 생성

일단 제일먼저 할 일은 Overlapped IO에 적합한 소켓을 생성하는 일이다. 그리고 이를 위해서는 다음 함수를 이용해야 한다.

```c
#include <winsock2.h>

SOCKET WSASocket(int f , int type , int protocol , LPWSAPROTOCOL_INFO lpProtocolInfo , GROUP g , DWORD dwFlags); // 성공 시 소켓의 핸들, 실패 시 INVALID_SOCKET 반환

// af : 프로토콜 체계 정보 전달
// type : 소켓의 데이터 전송방식에 대한 정보 전달
// protocol : 두 소켓 사이에 사용되는 프로토콜 정보 전달
// lpProtocolInfo : 생성되는 소켓의 특성 정보를 담고 있는 WSAPROTOCOL_INFO 구조체 변수의 주소 값 전달, 필요 없는 경우 NULL 전달
// g : 함수의 확장을 위해서 예약되어 있는 매개변수, 따라서 0 전달
// dwFlags : 소켓의 속성정보 전달
```

이 중에서 세번째 매개변수까지는 여러분이 잘 아는 것이다. 그리고 네 번째 매겹ㄴ수와 다섯번째 매개변수는 지금 우리가 하련느 일과 관계가 없으니, 각각 NULL과 0 을 전달해서, 생성되는 소켓에 Overlapped IO가 가능한 속성을 부여하자. 정리하면, 다음과 같이 소켓을 생성하면 이번 챕터에서 소개하는 Overlapped IO가 가능한 넌-블로킹 모드의 소켓이 생성된다.

```c
WSASocket(PF_INET , SOCK_STREAM , 0 , NULL , 0 , WSA_FLAG_OVERLAPPED);
```

## Overlapped IO를 진행하는 WSASend 함수

Overlapped IO 속성이 부여된 소켓의 생성 이후에 진행되는 두 소켓간의 연결과정은 일반소켓의 연결과정과 차이가 없다. 그러나 데이터의 입출력에 사용되는 함수는 달리해야 한다. 우선 Overlapped IO에 사용할 수 있는 데이터 출력함수를 먼저보이겠다.

```c
#include <winsock2.h>

WINSOCK_API_LINKAGE int WSAAPI WSASend(SOCKET s,LPWSABUF lpBuffers,DWORD dwBufferCount,LPDWORD lpNumberOfBytesSent,DWORD dwFlags,LPWSAOVERLAPPED lpOverlapped,LPWSAOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine); // 성공 시 0 , 실패 시 SOCKET_ERROR 반환

// s : 소켓의 핸들 전달, Overlapped IO 속성이 부여된 소켓의 핸들 전달 시 Overlapped IO 모델로 출력 진행
// lpBuffers : 전송할 데이터 정보를 지니는 WSABUF 구조체 변수들로 이뤄진 배열의 주소 값 전달
// dwBufferCount : 두번째 인자로 전달된 배열의 길이정보 전달
// lpNumberOfBytesSent : 전송된 바이트 수가 저장될 변수의 주소 값 전달
// dwFlags : 함수의 데이터 전송특성을 변경하는 경우에 사용, 예로 MSG_OOB를 전달하면 OOB 모드 데이터 전송
// lpOverlapped : WSAOVERLAPPED 구조체 변수의 주소 값 전달, Event 오브젝트를 사용해서 데이터 전송의 완료를 확인하는 경우에 사용되는 매개변수
// lpCompletionRoutine : Completion Routine 이라는 함수의 주소 값 전달, 이를 통해서도 데이터 전송의 완료를 확인할 수 있다.
```

이어서 위 함수의 두번째 인자로 전달되는 주소 갑스이 구조체를 소개하겠다. 이 구조체에는 전송할 데이터를 담고 있는 버퍼 주소 값과 크기 정보를 저장할 수 있도록 정의되어있다.

```c
typedef struct _WSABUF {
  u_long len;
  char *buf;
} WSABUF,*LPWSABUF;
```

그럼 구조체까지 설명하였으니, 위 함수의 호출형태를 간단히 보이겠다. 위 함수를 이용해서 데이터를 전송할 때에는 다음의 형태로 코드를 구성해야 한다.

```c
WSAEVENT event;
WSAOVERLAPPED overlapped;
WSABUF dataBuf;
char buf[BUF_SIZE] = "";
int recvBytes = 0;

event = WSACreateEvent();
memset(&overlapped , 0 , sizeof(overlapped));
overlapped.hEvent = event;
dataBuf.len = sizeof(buf);
dataBuf.buf = buf;
WSASend(hSocket , &dataBuf  1 , &recvBytes , 0 , &overlapped , NULL);
```

위의 WSASend 함수호출에서 세 번째 인자가 1인이유는 두번째 인자로 전달된 전송할 데이터를 담고 있는 버퍼의 정보가 하나이기 때문이다. 그리고 나머지 불필요한 인자에 대해서는 NULL, 또는 0를 전달하였는데, 여기서 여섯 번째 매개변수와 일곱번째 매개변수에 특히 주목하기 바란다. 여섯번째 인자로 전달된 구조체 WSAOVERLAPPED 은 다음과 같이 정의 되어있다.

```c
typedef struct _OVERLAPPED {
  ULONG_PTR Internal;
  ULONG_PTR InternalHigh;
  __C89_NAMELESS union {
      struct {
          DWORD Offset;
          DWORD OffsetHigh;
      } DUMMYSTRUCTNAME;
      PVOID Pointer;
  } DUMMYUNIONNAME;
  HANDLE hEvent;
} OVERLAPPED, *LPOVERLAPPED;
```

이 중에서 맴버 Internal , InternalHigh 는 Overlapped IO가 진행되는 과정에서 운영체제 내부적으로 사용되는 맴버이고, 맴버 Offset , OffsetHigh 역시 사용이 예약되어 있는 맴버이다. 때문에 여러분이 실제로 관심을 둘 맴버는 hEvent 가 전부이다. 이에 대한 활용방법도 잠시 후에 설명하기로 하고, 한가지만 더 이야기하고 다음으로 넘어가겠다.

"Overlapped IO를 진행하려면 WSASend 함수의 매개변수 lpOverlapped에는 항상 NULL 이 아닌 유효한 구조체 변수의 주소 값을 전달해야한다."

만약 lpOverlapped 에 NULL 이 전달되면 WSASend 함수의 첫번째 인자로 전달된 핸들의 소켓은 블로킹모드로 동작하는 일반적인 소켓으로 간주된다. 그리고 다음 사실도 기억하기 바란다. 이 사실을 몰라서 고생하는 경우가 많으니 말이다.

"WSASend 함수호출을 통해서 동시에 둘 이상의 영역으로 데이터를 전송하는 경우에는 여섯번째 인자로 전달되는 WSAOVERLAPPED 구조체 변수를 각각 별도로 구성해야 한다."

이는 WSAOVERLAPPED 구조체 변수가 Overlapped IO 진행과정에서 운영체제에 의해 참조되기 때문이다.

## WSASend 함수와 관련해서 한가지 더!

앞서 WSASend 함수의 매개변수 lpNumberOfBytesSent 를 통해서는 전송된 데이터의 크기가 저장된다고 설명했는데 이 부분이 좀 이상하지 않은가?

"WSASend 함수가 호출되자마자 반환하는데, 어떻게 전송된 데이터의 크기가 저장되나요?"

사실 WSASend 함수라고 해서 무조건 함수의 반환과 데이터의 전송완료 시간이 불일치하는 것은 아니다. 출력버퍼가 비어있고 전송하는 데이터의 크기가 크지 않다면 함수호출과 동시에 데이터의 전송이 완료될 수 있다. 그리고 이러한 경우에는 WSASend 함수가 0을 반환하고 매개변수 lpNumberOfBytesSent로 전달되는 주소의 변수에는 실제 전송된 데이터의 크기정보가 저장된다.

반면 호출된 WSASend 함수가 반환을 한 다음에도 계속해서 데이터의 전송이 이뤄지는 상황이라면 WSASend 함수는 SOCKET_ERROR 을 반환하고, WSAGetLastError 함수호출을 통해서 확인 가능한 오류코드로는 WSA_IO_PENDING이 등록된다. 그리고 이 경우에는 다음 함수호출을 통해서 실제 전송된 데이터의 크기를 확인해야 한다.

```c
#include <winsock2.h>

WINSOCK_API_LINKAGE WINBOOL WSAAPI WSAGetOverlappedResult(SOCKET s,LPWSAOVERLAPPED lpOverlapped,LPDWORD lpcbTransfer,WINBOOL fWait,LPDWORD lpdwFlags);
// 성공 시 TRUE , 실패시 FALSE 반환

// s : Overlapped IO가 진행된 소켓의 핸들
// lpOverlapped : Overlapped IO 진행 시 전달한 WSAOVERLAPPED 구조체 변수의 주소 값 전달
// lpcbTransfer : 실제 송수신된 바이트 크기를 저장할 변수의 주소 값전달.
// fWait : 여전히 IO가 진행중인 상황의 경우 , TRUE전달 시 IO가 완료될 때까지 대기를 하게 되고, FALSE 전달 시 FALSE 를 반환하면서 함수를 빠져나온다.
// lpdwFlag : WSARecv 함수가 호출된 경우 부수적인 정보를 얻기 위해 사용된다. 불필요하면 NULL 을 전달한다.
```

참고로 이 함수는 데이터의 전송결과뿐만 아니라, 데이터 수신결과의 확인에도 사용되는 함수이다. 왜냐하면 기능적으로 데이터를 전송하느냐 수신하느냐에 대한 차이만 있고, 나머지는 동일하기 때문이다.

## Overlapped IO를 진행하는 WSARecv 함수

WSASend 함수를 잘 이해했다면, WSARecv 함수는 쉽게 이해할 수 있다. 왜냐하면 기능적으로 데이터를 전송하느냐 수신하느냐에 대한 차이만 있고, 나머지는 동일하기 때문이다.

```c
#include <winsock2.h>

WINSOCK_API_LINKAGE int WSAAPI WSARecv(SOCKET s,LPWSABUF lpBuffers,DWORD dwBufferCount,LPDWORD lpNumberOfBytesRecvd,LPDWORD lpFlags,LPWSAOVERLAPPED lpOverlapped,LPWSAOVERLAPPED_COMPLETION_ROUTINE lpCompletionRoutine);
// 성공 시 0, 실패 시 SOCKET_ERROR 반환

// s : Overlapped IO 속성이 부여된 소켓의 핸들 전달
// lpBuffers : 수신된 데이터 정보가 저장될 버퍼의 정보를 지니는 WSABUF 구조체 배열의 주소 값 전달.
// dwBufferCount : 두 번째 인자로 전달된 배열의 길이정보 전달
// lpNumberOfBytesRecvd : 수신된 데이터의 크기정보가 저장될 변수의 주소 값 전달
// lpFlags : 전송특성과 관련된 정보를 지정하거나 수신하는 경우에 사용된다.
// lpOverlapped : WSAOVERLAPPED 구조체 변수의 주소 값 전달
// lpCompletionRoutine : Completion Routine 이라는 함수의 주소 값 전달
```

위 설명은 Overlapped IO기반의 데이터 입출력 방법이었다. 그러나 이후에 설명하는 내용은 IO의 완료 및 결과를 확인하는 방법이다.
