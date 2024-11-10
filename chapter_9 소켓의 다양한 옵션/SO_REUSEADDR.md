## SO_REUSEADDR
이번에 설명하는 SO_REUSEADDR 옵션 그리고 그와 관련있는 Time_wait 상태는 상대적으로 중요하다. 

## 주소할당 에러 (Binding error) 발생
SO_REUSEADDR 옵션에 대한 이해에 압서 Time-wait 사앹를 먼저 이해하는 것이 순서이다. 

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define TRUE 1
#define FALSE 0
void error_handling(char * message);

int main(int argc, char const *argv[])
{
  int serv_sock , clnt_sock;
  char message[30];
  int option , str_len;
  socklen_t optlen , clnt_adr_sz;
  struct sockaddr_in serv_adr , clnt_adr;

  
  if(argc != 2) {
    printf("Usage : %s <port> \n" , argv[0]);
    exit(1);
  }

  serv_sock = socket(PF_INET , SOCK_STREAM , 0);

  if(serv_sock == -1) 
    error_handling("socket() error");

  
  /*  
    optlen = sizeof(option);
    option = TRUE;
    setsockopt(serv_sock , SOL_SOCKET , SO_REUSEADDR , (void*) &option , optlen);
  */

  memset(&serv_adr , 0 , sizeof(serv_adr));
  serv_adr.sin_family = AF_INET;
  serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
  serv_adr.sin_port = htons(atoi(argv[1]));

  if(bind(serv_sock , (struct sockaddr *) &serv_adr , sizeof(serv_adr)))
    error_handling("bind() error");

  if(listen(serv_sock , 5) == -1)
    error_handling("listen error!");
  
  clnt_adr_sz = sizeof(clnt_adr);
  clnt_sock = accept(serv_sock , (struct sockaddr*) &clnt_sock , &clnt_adr_sz);

  while((str_len = read(clnt_sock , message , sizeof(message))) != 0) {
    write(clnt_sock , message , str_len);
    write(1 , message , str_len);
  }

  close(clnt_sock);
  close(serv_sock);
  return 0;
}

void error_handling(char * message) {
  fputs(message , stderr);
  fputc('\n' , stderr);
  exit(1);
}
```

위의 예제는 지금까지 몇 차례 구현한 에코 서버 프로그램이다. 

다음의 방식으로 프로그램을 종료해보자

"클라이언트 콘솔에서 Q 메시지를 입력하거나 CTRL+C를 입력해서 프로그램을 종료

즉, 클라이언트 측에서 서버 측으로 종료를 먼저 알리게끔 하라는 뜻이다. 클라이언트 콘솔에 Q 메시지를 입력하면 clsoe 함수를 호출하게 되어 서버 측으로 FIN 메시지를 먼저 전송하면서 Four-way handshaking 과정을 거치게 된다. 물론 CTRL+C 를 입력해도 동일하게 서버 측으로 FIN 메시지를 전달한다. 프로그램을 강제 종료할 경우에도 운영체제가 파일 및 소켓을 모두 닫아주는데, 이 과정에서 clsoe 함수를 호출한 것과 마찬가지로 서버 측으로 FIN 메시지가 전달된다.

만약 다음과 같이 프로그램을 종료한다면 어떻게 될까

"만약 서버와 클라이언트가 연결되 상태에서 서버 측 콘솔에서 CTRL+C를 입력한다. 즉 서버 프로그램을 강제 종료한다."

이는 서버가 클라이언트 측으로 먼저 FIN 메시지를 전달하는 상황의 연출을 위한것이다. 그런데 이렇게 서버를 종료하고 나면 서버의 재실행에 문제가 생긴다. 동일한 PORT번호를 기준으로 서버를 재실행하면 "bind() error" 라는 메시지가 출력될 뿐 서버는 실행되지 않는다. 그러나 이 상태에서 약 3분정도 지난 다음 재실행하면 정상적인 실행이 된다.

앞서 보인 두 가지 실행방식에 있어서의 유일한 차이점은 FIN 메시지를 누가 먼저 전송했는지에 있다.

## Time-wait 상태
먼저 Four-way handshking 과정을 살펴보자
![alt text](/image/image6.png)

호스트 A를 서버라고 보면 호스트 A가 호스트 B로 FIN 메시지를 먼저 보내고 있으니, 서버가 콘솔상에서 CTRL+C 를 입력한 상황으로 보자. 그런데 여기서 주목할 점은 연결의 해지 과정인 Four-way ahdnshaking 이후에 소켓이 바로 소멸되지 않고 Time-wait 상태라는 것을 일정시간 거친다는 점이다. 물론 Time-wait 상태는 먼저 연결의 종료를 요청한 호스트만 거친다. 이 때문에 서버가 먼저 연결의 종료를 요청해서 종료하고 나면 바로 이어서 실행 할 수 없는 것이다. 소켓이 Time-wait 상태에 있는 동안에는 해당 소켓의 PORT 번호가 사용중인 상태이기 때문이다. 따라서 앞서 확인한 것처럼 bind 함수의 호출과정에서 오류가 발생하는 것은 당연하다

## 클라이언트 소켓은 Time-wait 상태를 거치지 않나요?
Time-wait 상태는 서버에만 존재하는 것이 아닌 소켓의 Time-wait 상태는 먼저 연결의 종료를 요청하면 해당 소켓은 반드시 Time-wait 상태를 거친다. 그러나 클라이언트의 Time-wait 상태는 신경쓰지 않아도 된다. 왜냐하면 클라이언트 소켓의 Port 번호는 임의로 할당되기 때문이다. 즉, 서버와 달리 프로그램이 실행될 대 마다 PORT번호가 유동적으로 할당되기 때문에 Time-wait 상태에 대해 신경을 쓰지 않아도 된다.

그렇다면 Time-wait 상태는 무엇 때문에 존재할 까? 호스트 A가 호스트 B로 마지막 ACK 메시지(SEQ 5001 , ACK 7502)를 전송하고 나서 소켓을 바로 소멸시켰다고 가정하자. 그런데 이 마지막 ACK 메시지가 호스트 B로 전달되지 못하고 중간에 소멸되어 버렸다. 그렇다면 호스트 B는 자신이 좀 전에 보낸 FIN 메시지(SEQ 7501 , ACK 5001)가 호스트 A에 전송되지 못했다고 생각하고 재 전송을 시도할 것이다. 그러나 호스트 A의 소켓은 완전히 종료된 상태이기 때문에 호스트 B는 호스트 A로부터 영원히 마지막 ACK 메시지를 받지 못하게 된다. 반면 호스트 A 의 소켓이 Time-wait 상태로 놓였다면 호스트 B로 마지막 ACK 메시지를 재전송하게 되고 호스트 B는 정상적으로 종료가 가능하다 . 그러한 이유로 먼저 FIN 메시지를 전송한 호스트 소켓은 Time-wait 과정을 거치게 되는 것이다.

## 주소의 재할당
그러나 Time-wait 이 늘 반가운 것은 아니다. 시스템에 문제가 생겨 서버가 갑작스럽게 종료됬다고 가정하자. 재빨리 서버를 재 가동시켜서 서비스를 이어가야 하는데 Time-wait 상태 때문에 몇 분을 기다릴 수 밖에 없다면 이는 문제가 될 수 있다. 따라서 Time-wait 의 존재가 늘 반가울 수만은 없다. 또한 Time-wait 상태는 상황에 따라서 더 길어질 수 있어서 더 큰 문제로 이어질 수 있다. 

![alt text](/image/image7.png)

위 그림에서와 같이 호스트 A가 전송하는 Four-way handshaking 과정에서 마지막 데이터가 손실되면 호스트 B는 자신이 보낸 FIN 메시지를 호스트 A가 수신하지 못한 것으로 생각하고 FIN 메시지를 재 전송한다. 그러면 FIN 메시지를 수신한 호스트 A는 Time-wait 타이머를 재 가동한다. 때문에 네트워크의 상황이 원할하지 못한다면 Time-wait 상태가 언제까지 지속될 지 모른다.

## 해결책
소켓의 옵션 중에서 SO_REUSEADDR 의 상태를 변경하면 된다.
이의 적절한 변경을 통해서 Time-wait 상태에 있는 소켓에 할당되어 있는 PORT 번호를 새로 시작하는 소켓에 할당되게끔 할 수 있다. SO_REUSEADDR의 디폴트 값은 0(FALSE)인데 , 이는 Time-wait 상태에 있는 소켓의 PORT 번호는 할당이 불가능함을 의미한다. 따라서 이 값으 1(TRUE)로 변경해주어야 한다.

```c
 optlen = sizeof(option);
 option = TRUE;
 setsockopt(serv_sock , SOL_SOCKET , SO_REUSEADDR , (void*) &option , optlen);
```

주석을 해제하였는가? 이제 reuse_adr_eserver.c 는 언제건 실행가능한 상태가 되었으니 Time-wait 상태에서의 재실행이 가능한지 확인해보자

