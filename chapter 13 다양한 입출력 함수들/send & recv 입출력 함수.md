## 리눅스에서의 send & recv

```c
#include <sys/socket.h>

ssize_t send (int sockfd , const void * buf , size_t nbytes , int flags); // 성공시 전송된 바이트 수, 실패 시 -1 반환

// sockfd : 데이터 전송 대상과의 연결을 의미하는 소켓의 파일 디스크립터
// buf : 전송할 데이터 버퍼의 주소 값
// nbytes : 전송할 바이트 수 전달
// flags : 데이터 전송 시 전용할 다양한 옵션 정보 전달
```

```c
#include <sys/socket.h>

ssize_t recv(int sockfd , void * buf , size_t nbytes , int flags); // 성공 시 수신한 바이트 수 (단 EOF 시 0) , 실패 시 -1

// sockfd : 데이터 전송 대상과의 연결을 의미하는 소켓의 파일 디스크립터
// buf : 저장할 데이터 버퍼의 주소 값
// nbytes : 수신할 최대 바이트 수 전달
// flags : 데이터 수신 시 전용할 다양한 옵션 정보 전달
```

send 와 recv 의 마지막 매개변수에는 데이터 송수신시 적용할 옵션 정보가 전달된다. 옵션정보는 비트 OR 연산자( | 연산자 )를 이용해서 둘 이상을 함께 전달할 수 있다.

|옵션(Option)|의미|send|recv|
|:--:|:--:|:--:|:--:|
|MSG_OOB|긴급 데이터 (out-of-band data)의 전송을 위한 옵션|O|O|
|MSG_PEEK|입력 버퍼에 수신된 데이터의 존재 유무 확인을 위한 옵션||O|
|MSG_DONTROUTE|데이터 전송과정에서 라우팅(Routing) 테이블을 참조하지 않을 것을 요구하는 옵션|O||
|MSG_DONTWAIT|입출력 함수 호출과정에서 블로킹 되지 않을 것을 요구하기 위한 옵션 즉, 넌-브롤킹(Non-blocking) IO의 요구에 사용되는 옵션|O|O|
|MSG_WAITALL|요청한 바이트 수에 해당하는 데이터가 전부 수신될때까지 호출된 함수가 반환되는 것을 막기위한 옵션||O|

## MSG_OOB : 긴급 메시지의 전송
옵션 MSG_OOB 는 'out-of-band data'라 불리는 긴급 메시지의 전송에 사용된다.즉 , 긴급으로 무엇을 처리하려면 처리방법 및 경로가 달라야한다. 옵션 MSG_OOB는 긴급으로 전송해야할 메시지가 있어서 메시지의 전송방법 및 경로를 달리하고자 할 때 사용된다.

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#define BUF_SIZE 30
void error_handling(char * message);


int main(int argc, char const *argv[])
{
  int sock;
  struct sockaddr_in recv_adr;
  
  if(argc != 3) {
    printf("Usage : %s <IP> <port> \n" , argv[0]);
    exit(1);
  }

  sock = socket(PF_INET , SOCK_STREAM , 0);
  memset(&recv_adr , 0 , sizeof(recv_adr));
  recv_adr.sin_family = AF_INET;
  recv_adr.sin_addr.s_addr = inet_addr(argv[1]);
  recv_adr.sin_port = htons(atoi(argv[2]));

  if(connect(sock , (struct sockaddr*) &recv_adr , sizeof(recv_adr)) == -1) 
  error_handling("connect() error!");

  write(sock , "123" , strlen("123"));
  send(sock , "4" , strlen("4") , MSG_OOB);
  write(sock , "567" , strlen("567"));
  send(sock , "890" , strlen("890") , MSG_OOB);
  close(sock);

  return 0;
}


void error_handling(char * message) {
  fputs(message , stderr);
  fputc('\n' , stderr);
  exit(1);
}
```

```c
#define _XOPEN_SOURCE 200 // sigaction 에러 없애기
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <signal.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <fcntl.h>

#define BUF_SIZE 100
void error_handling(char * message);
void urg_handler(int signo);

int acpt_sock;
int recv_sock;

int main(int argc, char const *argv[])
{
  struct sockaddr_in recv_adr , serv_adr;
  int str_len , state;
  socklen_t serv_adr_sz;
  struct sigaction act;
  char buf[BUF_SIZE];

  if(argc != 2) {
    printf("Usage : %s <port> \n" , argv[0]);
    exit(1);
  }

  act.sa_handler = urg_handler;
  sigemptyset(&act.sa_mask);
  act.sa_flags = 0;

  acpt_sock = socket(PF_INET , SOCK_STREAM , 0);
  memset(&recv_adr , 0 , sizeof(recv_adr));
  recv_adr.sin_family = AF_INET;
  recv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
  recv_adr.sin_port = htons(atoi(argv[1]));

  if(bind(acpt_sock , (struct sockaddr *) &recv_adr , sizeof(recv_adr)) == -1) 
    error_handling("bind() error!");
  if(listen(acpt_sock , 5) == -1)
    error_handling("listen() error!");

  serv_adr_sz = sizeof(serv_adr);
  recv_sock = accpet(acpt_sock , (struct sockaddr*) &serv_adr , &serv_adr_sz);

  fcntl(recv_sock , __F_SETOWN , getpid());
  state = sigaction(SIGURG , &act , 0);

  while((str_len = recv(recv_sock , buf , sizeof(buf) , 0)) != 0) {
    if(str_len = -1) 
      continue;
    buf[str_len] = 0;
    puts(buf);
  }

  close(recv_sock);
  close(acpt_sock);

  return 0;
}

void urg_handler(int signo) {
  int str_len;
  char buf[BUF_SIZE];
  str_len = recv(recv_sock , buf , sizeof(buf) -1 , MSG_OOB);
  buf[str_len] = 0;
  printf("Urgent message : %s \n" , buf);
}

void error_handling(char * message) {
  fputs(message , stderr);
  fputc('\n' , stderr);
  exit(1);
}

```

위 예제에는 지금까지 설명하지 않은 함수가 있다.

```c
fcntl(recv_sock , F_SETOWN , getpid());
```

fcntl 함수는 파일 디스크립터의 컨트롤에 사용이 된다. 위의 문장은 다음의 의미를 지닌다.

"파일 디스크립터 recv_sock 이 가리키는 소켓의 소유자 (F_SETOWN)를 getpid()함수가 반환하는 ID의 프로세스로 변경시키겠다."

소켓의 소유자라는 개념이 다소 생서하게 느껴질 거싱다. 사실 소켓은 운영체제가 생성 및 관리를 하기 때문에 엄밀히 말하면 소켓의 소유자는 운영체제이다. 다만 여기서 말하는 소유자는 이 소켓에서 발생하는 모든 일의 책임 주체를 의미한다. 위의 예제 상황을 쉽게 설명하면 다음과 같다

"파일 디스크립터 recv_sock 이 가리키는 소켓에 의해 발생하는 SIGURG 시그널을 처리하는 프로세스를 getpid 함수가 반환하는 ID의 프로세스로 변경시키겠다.

물론 위 문장에서의 SIGURG 시그널 처리는 'SIGURG 시그널의 핸들러 함수호출'을 의미한다. 그런데 하나의 소켓에 대한 파일 디스크립터를 여러 프로세스가 함께 소유할 수 있지 않은가? 예를 들어서 fork 함수 호출을 통해서 자식 프로세스가 생성되고 생성과 동시에 파일 디스크립터까지 복사되는 경우도 이에 해당한다. 이러한 상황에서 SIGURG 시그널 발생시 어느 프로세스의 핸들러 함수를 호출해야 하겠는가? 모든 프로세스의 핸들러 함수가 호출되지 않는다.

따라서 SIGURG 시그널을 핸들링 할 때에는 반드시 시그널을 처리할 프로세스를 지정해 줘야 한다. 그리고 getpid는 이 함수를 호출한 프로세스의 ID를 반환하는 함수이다. 결국 위의 문장은 현재 실행중인 프로세스를 SIGURG 시그널의 처리 주체로 지정하는 것이다.

## 출력 결과
```
123
Urgent message : 4
567
Urgent message : 0
89
```

"뭐야 MSG_OOB 옵션을 추가해서 데이터를 전달할 경우 딱 1바이트만 반환이 되잖아? 그리고 뭐 특별히 빨리 전송된 것도 아니네!"

아쉽지만 MSG_OOB 옵션을 추가한다고 해서 더 빨리 데이터가 전송되는 것은 아니고 시그널 핸들러인 urg_handler 함수를 통해서 읽히는 데이터도 1바이트밖에 되지 않는다. 나머지 MSG_OOB 옵션이 추가되지는 않은 일반적인 입력함수의 호출을 통해서 읽히고 만다. 왜냐하면 TCP에는 진정한 의미의 'out-of-band data'가 존재하지 않기 때문이다. 사실 MSG_OOB 에서의 OOB 는 out-of-band 를 의미한다 그리고 이는 다음의 의미를 지닌다.

"전혀 다른 통신 경로로 전송되는 데이터"

즉, out-of-band 형태로 데이터가 전송되려면 별도의 통신 경로가 확보되어서 고속으로 데이터가 전달되어야 한다. 하지만 TCP 는 별도의 통신 경로를 제공하지 않고 있다. 다만 TCP에 존재하는 Urgent mode 라는 것을 이용해서 데이터를 전송해줄 뿐이다.
