## TCP 기반 서버 , 클라이언트 구현
드디어 TCP 기반의 서버를 완벽하게 구현해보자

## TCP 서버에서의 기본적인 함수호출 순서
1. socket() : 소켓생성
2. bind() : 소켓 주소 할당
3. listen() : 연결요청 대기 상태
4. accept() : 연결 허용
5. read() / write() : 데이터 송수신
6. close() : 연결종료

제일먼저 socket 함수의 호출을 통해서 소켓을 생성한다. 그리고 주소정보를 담기 위한 구조체 변수 선언 및 초기화해서 bind 함수를 호출해서 소켓에 주소를 할당한다. 이 두 단계는 이미 했으니 그 이후의 과정에 대하여 알아보자


## 연결요청 대기 상태로의 진입
bind 함수호출을 통해서 소켓에 주소까지 할당했다면, 이번에는 listen 함수호출을 통해서 '연결요청 대기상태'로 들어갈 차례이다. 그리고 listen 함수가 호출되어야 클라이언트가 연결 요청을 할 수 있는 상태가 된다. 즉, listen 함수가 호출되어야 클라이언트는 연결요청을 위해서 connect 함수를 호출할 수 있다. (이전에 connect 함수가 호출되면 오류가 발생)

```c
#include <sys/socket.h>

int listen(int sock , int backlog); // 성공 시 0 , 실패시 -1

// sock : 연결요청 대기상태에 두고자하는 소켓의 파일 디스크립터 전달, 이 함수의 인자로 전달된 디스크립터의 소켓이 서버 소켓(리스닝 소켓) 이 된다.
// backlog : 연결요청 대기 큐(Queue)의 크기정보 전달, 5가 전달되면 큐의 크기가 5가 되어 클라이언트 연결요청을 5개까지 대기시킬 수 있다.
```

여기서 잠시 '연결요청 대기상태' 의 의미와 '연결요쳥 대기 큐'라는 것에 대해 별도의 설명을 해본다. 서버가 '연결요청 대기상태'에 있다는 것은 클라이언트가 연결요청을 했을대 연결이 수락될 때까지 연결요청 자체를 대기시킬 수 있는 상태에 있다는 것을 의미한다.

![alt text](https://mblogthumb-phinf.pstatic.net/MjAxOTAzMjRfOSAg/MDAxNTUzNDA1MjY5MjA5.4pNvEoF1q0rQzRmkJ0UogrfBS5NY-uMv_RNdd0DsZtQg.3SqiBeWANUy-T6cHNk3ZNn7pkdlCzUA2QTN-hERoIbMg.PNG.ihp0001/image.png?type=w800)

위 그림을 보면 listen 함수의 첮번째 인자로 전달된 파일 디스크립터의 소켓이 어떤 용도로 사용되는 지를 알 수 있다. 클라이언트 연결요청도 인터넷을 통해서 흘러들어오는 일종의 데이터 전송이기 때문에 이것을 받아들이려면 당연히 소켓이 하나 있어야한다. 서버 소켓의 역할이 바로 이것이다. 즉, 연결 요청을 맞이하는 일종의 문지기 또는 문의 역할을 한다고 볼 수 있다. 클라이언트가 '저기여 혹시 제가 감히 연결될 수 있나요?' 라고 서버 소켓에게 물어보면 서버 소켓은 아주 친절한 문지기이기때문에 '아 물론이죠. 그런데 지금 시스템이 조금 바쁘니, 대기실에서 번호표 뽑고 기다리시면 준비되는 대로 바로 연결해 드리겠습니다.' 라고 말하며 클라이언트의 연결요청을 대기실로 안내한다. listen 함수가 호출되면 이렇듯 문지기의 역할을 하는 서버 소켓이 만들어지고, listen 함수의 두번째 인자로 전달되는 정수의 크기에 해당하느 대기실이 만들어 진다.

이 대기실을 가리켜 '연결요청 대기 큐' 라고 하며 서버 소켓과 연결요청 대기 큐가 완전히 준비되어서 클라이언트의 연결 요청을 받아들일 수 있는 상태를 가리켜 '연결요청 대기 상태' 라 한다.

listen 함수의 두번째 인자로 전달될 적절한 인자의 값은 서버의 성격마다 다르지만, 웹 서버와 같이 잦은 연결요청을 받는 서버의 경우 최소 15이상을 전달해야 한다. 참고로 연결요청 대기 큐의 크기는 어디까지나 실험적 결과에 의존해서 결정된다.

## 클라이언트 연결요청 수락
listen ㅎ마수호출이후 클라이언트의 연결요청이 들어왔다면 들어온 순서대로 연결요청을 수락해야한다. 연결요청을 수락한다는 것은 클라이언트와 데이터를 주고받을 수 있는 상태가 됨을 의미한다. 따라서 이러한 상태가 되기 위해 무엇이 필요할까? 당연히 소켓이다!

전혀 이상할 것이 없다. 데이터를 주고받으려면 소켓이 있어야 하며, 물론 우리는 서버 소켓을 생각하면 이것을 사용하면되지 않느냐고 물을 수 있다. 그런데 서버 소켓은 문지기이다. 클라이언트와의 데이터 송수신을 위해 이것을 사용하면 문은 누가 지키겠는가? 때문에 소켓을 하나 더 만들어야 한다. 하지만 우리가 소켓을 직접 만들 필요는 없다. 다음 함수의 호출결과로 소켓이 만들어지고, 이 소켓은 열결 요청을 한 클라이언트 소켓과 자동으로 연결되니 말이다.

```c
#include <sys/socket.h>

int accept(int sock , struct sockaddr* addr , socklen_t * addrlen) // 성공시 생선된 소켓의 파일 디스크립터 , 실패시 -1 반환

// sock : 서버 소켓의 파일 디스크립터 전달
// addr : 연결요청 한 클라이언트의 주소정보를 담을 변수의 주소 값 전달, 함수호출이 완료되면 인자로 전달된 주소의 변수에는 클라이언트 주소정보가 채워진다.
// addrlen : 두번째 매개변수 addr 에 전달된 주소의 변수 크기를 바이트 단위로 전달, 단 크기정보를 변수에 저장한 다음에 변수의 주소 값을 전달한다. 그리고 함수호출이 완료되면 크기정보로 채워져 있던 변수에는 클라이언트의 주소정보 길이가 바이트 단위로 계산되어 채워진다.
```

accpet 함수는 '연결 요청 대기 큐' 에서 대기중인 클라이언트의 연결요청을 수락하는 기능의 함수이다. 따라서 accept 함수는 호출성공 시 내부적으로 데이터 입출력에 사용할 소켓을 새엇ㅇ하고, 그 소켓의 파일 디스크립터를 반환한다. 중요한 점은 소켓이 자동으로 생성되어, 연결요청을 한 클라이언트 소켓에 연결까지 이뤄진다는 점이다.

![alt text](https://mblogthumb-phinf.pstatic.net/MjAxOTAzMjRfOSAg/MDAxNTUzNDA1MjY5MjA5.4pNvEoF1q0rQzRmkJ0UogrfBS5NY-uMv_RNdd0DsZtQg.3SqiBeWANUy-T6cHNk3ZNn7pkdlCzUA2QTN-hERoIbMg.PNG.ihp0001/image.png?type=w800)

대기 큐(Queue) 에 존재하던 연결 요청 하나를 꺼내서 새로운 소켓을 생성한 후에 연결 요청을 완료함을 보이고 있다. 이렇듯 서버에서 별로도 생성한 소켓과 클라이언트 소켓이 직접 연결되었으니, 이제는 데이터를 주고받는 일만 남았다.

## TCP 클라이언트의 기본적인 함수호출 순서
이번에는 클라이언트의 구현순서에 대해서 이야기 해보겠다.

1. socket() : 소켓생성
2. connect() : 연결요청
3. read() / write() : 데이터 송수신
4. close() : 연결종료

서버의 구현과정과 비교해서 차이가 있는 부분은 '연결요청' 이라는 과정이다. 이는 클라이언트 소켓을 생성한 후에 서버로 연결을 요청하는 과정이다. 서버는 listen 함수를 호출한 이후부터 연결요청 대기 큐를 만들어 놓는다. 따라서 그 이후부터 클라이언트는 연결요청을 할 수 있다. 그렇다면 클라이언트는 어떻게 연결요청을 할까?

```c
#include <sys/socket.h>

int connect(int sock , struct sockaddr* servaddr , socklen_t addrlen) // 성공 시 0 , 실패 시 -1 반환

// sock : 클라이언트 소켓의 파일 디스크립터 전달
// servaddr : 연결요청 할 서버의 주소정보를 담은 변수의 주소 값 전달
// addrlen : 두 번째 매겨변수 servaddr 에 전달된 주소의 변수 크기를 바이트 단위로 전달
```

클라이언트에 의해서 connect 함수가 호출되면 다음 둘 중 한가지 상황이 되어야 함수가 반환된다. (함수 호출이 완료된다.)

1. 서버에 의해 연결요청이 접수되었다.
2. 네트워크 단절 등 오류상황이 발생해서 연결요청이 중단되었다.

여기서 주의할 사실은 위에서 말하는 '연결요청의 접수' 는 서버의  accept 함수호출을 의미하는 것이 아니라는 점이다. 이는 클라이언트의 연결요청 정보가 서버의 연결요청 대기 큐에 등록된 상황을 의미하는 것이다. 때문에 connect 함수가 반환했더라도 당장에 서비스가 이뤄지지 않을 수 있음을 기억해야 한다.

## TCP 기반 서버, 클라이언트 함수호출관계
지금까지 TCP 서버, TCP 클라이언트 프로그램의 구현 순서를 알아보았다.