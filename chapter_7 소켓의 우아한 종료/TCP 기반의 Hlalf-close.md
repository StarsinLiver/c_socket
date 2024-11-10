## TCP 기반의 Half-close
TCP 에서는 연결과정보다 중요한 것이 종료과정이다. 연결과정에서는 큰 변수가 발생하지 않지만 종료 과정에서는 예상치 못한 일이 발생할 수 있기 때문에 종료과정은 명확해야 한다.

## 일방적인 연결종료의 문제점
리눅스의 clsoe 함수호출과 윈도우의 closesocket 함수 호출은 완전종료를 의미한다. 완전종료라는 것은 데이터를 전송하는 것은 물론이거니와 수신하는것 조차 더 이상 불가능한 상황을 의미한다. 때문에 한쪽에서의 일방적인 clsoe 또는 closesocket 함수 호출은 경우에 따라서 우아해 보이지 못할 수 있다.

호스트 A가 마지막 데이터를 전송하고 나서 close 함수의 호출을 통해서 연결을 종료하였다. 때문에 그 이후부터 호스트 A는 호스트 B가 전송하는 데이터를 수신하지 못한다. 아니! 데이터 수신과 관련된 함수의 호출 자체가 불가능하다. 때문에 결국엔 호스트 B가 전송한 호스트 A가 반드시 수신해야 할 데이터라 할지라도 그냥 소멸되고 만다.

이러한 문제의 해결을 위해서 데이터의 송수신에 사용되는 스트림의 일부만 종료 (Half-close) 하는 방법이 제공되고 있다. 일부를 종료한다는 것은 전송은 가능하지만 수신은 불가능한 상황, 혹은 수신은 가능하지만 전송은 불가능한 상황을 뜻한다. 말 그대로 스트림의 반만 닫는 것이다.

## 소켓 과 스트림 (stream)
소켓을 통해서 두 호스트가 연결되면, 그 다음부터는 상호간에 데이터의 송수신이 가능한 상태가 된다.
그리고 바로 이러한 상태를 가리켜 '스트림이 형성된 상태' 란 한다.즉, 두 소켓이 연결되어서 데이터의 송수신이 가능한 상태를 일종의 스트림으로 보는 것이다.

스트림은 물의 흐름을 의미한다. 그런데 물의 흐름은 한쪽 방향으로만 형성된다. 마찬가지로 소켓의 스트림 역시 한쪽 방향으로만 데이터의 이동이 가능하기 때문에 양방향 통신을 위해서는 다음 그림에서 보이듯이 두 개의 스트림이 필요하다.

![alt text](/image/image5.png)

때문에 두 호스트간에 소켓이 연결되면 각 호스트 별로 입력 스트림과 출력 스트림이 형성된다

## 우아한 종료를 위한 shutdown 함수
그럼 이제 우아한 종료 즉, Half-close 에 사용되는 함수를 알아보자

```c
#include <sys/socket.h>

int shutdown(int sock , int howto); // 성공 시 0 , 실패 시 -1 반환
// sock : 종료할 소켓의 파일 디스크립터 전달
// howto : 종료방법에 대한 정보 전달
```

위의 함수 호출시 두 번째 매개변수에 전달되는 인자에 따라서 종료의 방법이 결정된다. 두 번째 매개변수에 전달될 수 있는 인자의 종류는 다음과 같다.

```
1. SHUT_RD : 입력 스트림 종료
2. SHUT_WR : 출력 스트림 종료
3. SHUT_RDWR : 입출력 스트림 종료
```

shutdown 함수의 두번째 인자로 SHUT_RD 가 전달되면 입력 스트림이 종료되어 더 이상 데이터를 수신할 수 없는 상태가 된다. 혹 데이터가 입력버퍼에 전달되더라도 그냥 지워져 버릴 뿐만 아니라 입력 관련 함수의 호출도 더 이상은 허용이 안된다. 반면 SHUT_WR 가 두 번재 인자로 전달되면 출력 스트림이 종료되어 더 이상의 데이터 전송이 불가능해진다. 단! 출력 버퍼에 아직 전송되지 못한 상태로 남아있는 데이터가 존재하면 해당 데이터는 목적지로 전송된다. 마지막으로 SHUT_RDWR가 전달되면 입력 스트림 , 출력 스트림이 모두 종료된다.

## Half-close 가 필요한 이유
소켓은 반만 닫는 것에 대한 의미는 충분히 이해했다. 그러나 아직 풀어야할 궁금증이 남는다.

"half-close 가 도대체 왜 필요한거지 그냥 데이터를 주고받기에 충분한 만큼 연결을 유지했다가 종료하면 되는것 아닌가? 급히 종료하지 않으면 Half-close 가 필요하지는 않을 건데 말이야!"

이에 대한 상황을 생각해보자

"클라이언트가 서버에 접속하면 서버는 약속된 파일을 클라이언트에게 전송하고, 클라이언트는 파일을 잘 수신했다는 의미로 문자열 "Thank you" 를 서버에 전송한다."

여기서 문자열 "Thank you"의 전달은 사실상 불필요한 일이지만, 연결 종료 직전에 클라이언트가 서버에 전송해야할 데이터가 존재하는 상황으로 확대해석하기 바란다. 그런데 이 상황에 대한 프로그램의 구현도 그리 간단하지는 않다. 파일을 전송하는 서버는 단순히 파일 데이터를 연속해서 전송하면 되지만, 클라이언트는 언제까지 데이터를 수신해야 할 지 알수 없기 때문이다. 클라이언트 입장에서는 무턱대고 계속해서 입력함수를 호춣 할 수도 없는 노릇이다. 그랬다가는 블로킹 상태에 빠질 수 있다.

"서버와 클라이언트 사이에 파일의 끝을 의미하는 문자 하나를 약속하면 되지 않는가?"

이것도 어울리지 않는 상황이다. 약속으로 정해진 문자와 일치하는 데이터가 파일에 존재할 수 있기 때문이다.

이러한 문제를 해결하지 위해서 서버는 파일의 전송이 끝났음을 알리는 목적으로 EOF를 마지막에 전송해야 한다.

"출력 스트림을 종료하면 상대 호스트로 EOF 가 전송됩니다."

물론 clsoe 함수호출을 통해서 입출력 스트림을 모두 종료해줘도 EOF는 전송되지만, 이럴 경우 상대방이 전송하는 데이터를 더 이상 수신 못한다는 문제가 있다. 즉, close 함수 호출을 통해서 스트림을 종료하면 클라이언트가 마지막으로 보내는 문자열 "Thank you" 를 수신할 수 없다. 따라서 shutdown 함수 호출을 통해서 서버의 출력 스트림만 Half-close 해야 하는 것이다. 이럴 경우 EOF도 전송되고 입력 스트림은 여전히 살아있어서 데이터의 수신도 가능하다.

## Half-close 기반의 파일 전송 프로그램
```c
// server
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#define BUF_SIZE 1024
void error_handling(char * message);

int main(int argc, char const *argv[])
{
  int serv_sd , clnt_sd;
  FILE * fp;
  char buf[BUF_SIZE];
  int read_cnt;

  struct sockaddr_in serv_adr , clnt_adr;
  socklen_t clnt_adr_sz;

  if(argc != 2) {
    printf("Usage : %s <port> \n" , argv[0]);
    exit(1);
  }

  fp = fopen("./file_server.c" , "rb");
  serv_sd = socket(PF_INET , SOCK_STREAM , 0);

  memset(&serv_adr , 0 , sizeof(serv_adr));
  serv_adr.sin_family = AF_INET;
  serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
  serv_adr.sin_port = htons(atoi(argv[1]));

  bind(serv_sd , (struct sockaddr *) &serv_adr , sizeof(serv_adr));
  listen(serv_sd , 5);

  clnt_adr_sz = sizeof(clnt_adr);
  clnt_sd = accept(serv_sd , (struct sockaddr *) & clnt_adr , &clnt_adr_sz);

  while(1) {
    read_cnt = fread((void*) buf , 1 , BUF_SIZE , fp);
    if(read_cnt < BUF_SIZE) {
      write(clnt_sd , buf , read_cnt);
      break;
    }

    write(clnt_sd , buf , BUF_SIZE);
  }

    shutdown(clnt_sd , SHUT_WR);
    read(clnt_sd , buf , BUF_SIZE);
    printf("Message from client , %s \n" , buf);

    fclose(fp);
    close(clnt_sd) ; close(serv_sd);
  return 0;
}


void error_handling(char * message) {
  fputs(message , stderr);
  fputc('\n' , stderr);
  exit(1);
}
```

```c
// client 
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#define BUF_SIZE 1024
void error_handling(char * message);

int main(int argc, char const *argv[])
{
  int sd;
  FILE * fp;
  char buf[BUF_SIZE];
  int read_cnt;
  struct sockaddr_in serv_adr;

  if(argc != 3) {
    printf("Usage : %s <IP> <port> \n" , argv[0]);
    exit(1);
  }

  fp = fopen("receive.dat" , "wb");
  sd = socket(PF_INET, SOCK_STREAM, 0);

  memset(&serv_adr , 0 , sizeof(serv_adr));
  serv_adr.sin_family = AF_INET;
  serv_adr.sin_addr.s_addr = inet_addr(argv[1]);
  serv_adr.sin_port = htons(atoi(argv[2]));

  connect(sd , (struct sockaddr *) &serv_adr , sizeof(serv_adr));

  while((read_cnt = read(sd , buf , BUF_SIZE)) != 0) 
    fwrite((void*) buf , 1 , read_cnt , fp);

  puts("Received file data");
  write(sd , "Thank you" , 10);
  fclose(fp);
  close(sd);
  
  return 0;
}


void error_handling(char * message) {
  fputs(message , stderr);
  fputc('\n' , stderr);
  exit(1);
}
```

