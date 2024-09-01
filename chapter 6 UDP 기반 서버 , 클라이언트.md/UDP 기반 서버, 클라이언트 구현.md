## UDP 에서의 서버와 클라이언트는 연결되어 있지 않습니다,
UDP 서버 , 클라이언트는 TCP와 같이 연결된 상태로 데이터를 송수신하지 않는다. 때문에 TCP와 달리 연결 설정의 과정이 필요없다. 따라서 TCP 서버 구현과정에서 거쳤던 LISTEN 함수와 accept 함수의 호출은 필요없다.

## UDP 에서는 서버건 클라이언트건 하나의 소켓만 있으면 됩니다.
TCP 에서는 소켓과 소켓의 관계가 일대일이다. 때문에 서버에서 열 개의 클라이언트에게 서비스를 제공하려면 문지기의 역할을 하는 서버 소켓을 제외하고도 열 개의 소켓이 더 필요했다. 그러나 UDP는 서버건 클라이언트건 하나의 소켓만 있으면 된다. 앞서 UDP 를 말할 때 편지를 예로 들었는데, 편지를 주고받기 위해서 필요한 우체통을 UDP 소켓에 비유할 수 있다. 우체통이 근처에 하나 있다면 이를 이용해서 어디건 편지를 보낼 수 있지 않는가? 마찬가지로 UDP 소켓이 하나 있다면 어디건 데이터를 전송할 수 있다.

## UDP 기반의 데이터 입출력함수
TCP 소켓을 생성하고 나서 데이터를 전송하는 경우에는 주소 정보를 따로 추가하는 과정이 필요없다. 왜냐하면 TCP 소켓은 목적지에 해당하는 소켓과 연결된 상태이기 때문이다. 즉, TCP 소켓은 목적지의 주소정보를 이미 알고있는 상태다. 그러나 UDP 소켓은 연결상태를 유지하지 않으므로 데이터를 전송항때마다 반드시 목적지의 주소정보를 별도로 추가해야한다. 이는 우체통에 넣은 우편물에 주소정볼르 써 넣는 것에 비유할 수 있다. 그럼 여기 주소정보를 넣으면서 데이터를 전송할 대는 호출하는 UDP 관련 함수를 소개하겠다.

```c
#include <sys/socket.h>

ssize_t sendto(int sock , void *buff , size_t nbytes , int flags , struct sockaddr * to , socklen_t addrlen);

// sock : 데이터 전송에 사용될 UDP 소켓의 파일 디스크립터를 인자로 전달
// buff : 전송할 데이터를 저장하고 있는 버퍼의 주소 값 전달
// nbytes : 전송할 데이터 크기를 바이트 단위로 전달.
// flags : 옵션 지정에 사용되는 매개변수 , 지정할 옵션이 없다면 0 전달
// to    : 목적지 주소정보를 담고 있는 sockaddr 구조체 변수의 주소 값 전달
// addrlen : 매개변수 to로 전달된 주소 값의 구조체 변수 크기를 전달
```

위 함수가 이전 소개한 TCP 기반의 출력함수와 가장 비교되는 것은 목적지 주소정보를 요구하고 있다는 점이다. 이어서 UDP 데이터 수신에 사용되는 함수를 소개하겠다. UDP 데이터는 발신지가 일정치 않기 때문에 발신지 정볼르 얻을 수 있도록 함수가 정의되어있다. 즉, 이 함수는 UDP 패킷에 담겨있는 발신지 정보를 함께 반환한다.

```c
#include <sys/socket.h>

ssize_t recvfrom(int sock , void *buff , size_t nbytes , int flags , struct sockaddr *from , socklen_t *addrlen); // 성공시 수신한 바이트 수 , 실패 시 -1 반환

// sock : 데이터 수신에 사용될 UDP 소켓의 파일 디스크립터를 인자로 전달
// buff : 데이터 수신에 사용될 버퍼의 주소 값 전달.
// nbytes : 수신할 최대 바이트 수 전달, 때문에 매개변수 buff 가 가리키는 버퍼의 크기를 넘을 수 없다.
// flags : 옵션 지정에 사요오디는 매개변수 , 지정할 옵션이 없다면 0 전달.
// from : 발신지 정보를 채워넣을 sockaddr 구조체 변수의 주소 값 전달.
// addrlen : 매개변수 from 으로 전달된 주소에 해당하는 구조체 변수의 크기정보를 담고 있는 변수의 주소값 전달
```

UDP 프로그램 작성의 핵심은 지금 위의 두 함수에 있다고 해도 과언이 아니다. 그 정도로 UDP 관련 데이터 송수신에서 이 두 함수가 차지하는 위치는 매우 크다

## UDP 기반의 에코 서버와 에코 클라이언트
에코 서버를 구현해보자 참고로 UDP 에서는 TCP 와 달리 연결요청과 그에 따른 수락의 과정이 존재하지 않기 대문에 서버 및 클라이언트라는 표현이 적절치 않은 부분도 있다. 다만 서비스를 제공한다는 측면에서 서버라 하는 것이다.


```c
// 서버 
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char * message);

int main(int argc, char const *argv[])
{
  int serv_sock;
  char message[BUF_SIZE];
  int str_len;
  socklen_t clnt_adr_sz;

  struct sockaddr_in serv_adr , clnt_adr;

  if(argc != 2) {
    printf("Usage : %s <port> \n" , argv[0]);
    exit(1);
  }

  serv_sock = socket(PF_INET , SOCK_DGRAM , 0);
  if(serv_sock == -1) 
    error_handling("UDP socket creation error!");

  memset(&serv_adr , 0 , sizeof(serv_adr));
  serv_adr.sin_family = AF_INET;
  serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
  serv_adr.sin_port = htons(atoi(argv[1]));

  if(bind(serv_sock , (struct sockaddr * ) &serv_adr , sizeof(serv_adr)) == -1) 
    error_handling("bind() error");

  while(1) {
    clnt_adr_sz = sizeof(clnt_adr);
    str_len = recvfrom(serv_sock , message , BUF_SIZE , 0 , (struct sockaddr*) &clnt_adr , &clnt_adr_sz);
    sendto(serv_sock , message , str_len , 0 , (struct sockaddr *) &clnt_adr , clnt_adr_sz);
  }

  close(serv_sock);
  return 0;
}

void error_handling(char * message) {
  fputs(message , stderr);
  fputc('\n' , stderr);
  exit(1);
}
```

```c
// 클라이언트
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char * message);

int main(int argc, char const *argv[])
{
  int sock;
  char message[BUF_SIZE];
  int str_len;
  socklen_t adr_sz;

  struct sockaddr_in serv_adr , from_adr;

  if(argc != 3) {
    printf("Usage : %s <IP> <port> \n" , argv[0]);
    exit(1);
  }

  sock = socket(PF_INET , SOCK_DGRAM , 0);
  if(sock == -1)
    error_handling("socket() error");

  memset(&serv_adr , 0 , sizeof(serv_adr));
  serv_adr.sin_family = AF_INET;
  serv_adr.sin_addr.s_addr = inet_addr(argv[1]);
  serv_adr.sin_port = htons(atoi(argv[2]));

  while(1) {
    fputs("Insert message (q to Quit) : " , stdout);
    fgets(message , sizeof(message) , stdin);
    if(!strcmp(message , "q\n") || !strcmp(message , "Q\n")) 
      break;

    sendto(sock , message , strlen(message) , 0 , (struct sockaddr*) &serv_adr , sizeof(serv_adr));
    adr_sz = sizeof(from_adr);

    str_len = recvfrom(sock , message , BUF_SIZE , 0 , (struct sockaddr *) &from_adr , &adr_sz);
    message[str_len] = 0;
    printf("Message from server %s" , message);
  }

  close(sock);
  return 0;
}

void error_handling(char * message) {
  fputs(message , stderr);
  fputc('\n' , stderr);
  exit(1);
}
```

## UDP 클라이언트 소켓의 주소정보 할당
지금까지 UDP 기반의 서버, 클라이언트 프로그램 구현에 대해 설명하였다. 그런데 UDP 클라이언트 프로그램을 가만히 보면 , IP 와 PORT 를 소켓에 할당하는 부분이 눈에 띄지 않는다.  TCP 클라이언트의 경우에는 connect 함수 호출시 자동으로 할당되었는데, UDP 클라이언트의 경우에는 그러한 기능을 대신 할만한 함수호출문 조차 보이지 않는다. 도대체 어느 시점에 IP 와 PORT가 할당되는 것일까?

udp 프로그램에서는 데이터를 전송하는 sendto 함수호출 이전에 해당 소켓에 주소정보가 할당되어있어야 한다. 따라서 sendto 함수호출 이전에 해당 소켓에 주소정보가 할당되어 있어야 하는데 따라서 bind 함수를 통해 주소정보를 할당해야한다. 물론 bind 함수는 TCP 프로그램의 구현에서 호출되었던 함수이다. 그러나 이 함수는 TCP 와 UDP 를 가리키지 않으므로 UDP 프로그램에서도 호출 가능하다. 그리고 만약에 sendto 함수 호출시 까지 주소정보가 할당되지 않았다면 sendto 함수가 처음 호출되는 시점에 해당 소켓에 IP와 PORT 번호가 자동으로 할당된다.

또한 이렇게 한번 할당되면 프로그램이 종료될 때까지 주소정보가 그대로 유지되기 때문에 다른 UDP 소켓과 데이터를 주고받을 수 있다. 물론 IP 는 호스트의 IP 로 , PORT는 사용하지 않는 PORT번호 하나를 임의로 골라서 할당하게 된다.

이렇듯 sendto 함수호출 시 IP 와 PORT 번호가 자동으로 할당되기 때문에 일반적으로 UDP의 클라이언트 프로그램에서는 주소정보를 할당하는 별도의 과정이 불필요하다. 그래서 앞서 보인 예제에서도 별도의 주소정보 할당과정을 생략했으며 이것이 일반적인 구현방법이다.
