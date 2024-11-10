## UDP의 데이터 송수신 특성과 UDP에서의 connect 함수호출
TCP 기반에서 송수신하는 데이터에는 경계가 존재하지 않음을 앞서 충분히 설명하였고, 이를 예제를 통해서 확인까지 하였다. 그래서 이번에는 UDP 기반에서 송수신하는 데이터에 경계가 존재함을 예제를 통해서 확인하고자한다.

그리고 UDP 에서의 connect 함수호출과 관련해서 조금 더 이야기하면서 UDP 에 대한 설명을 마무리 해보자

## 데이터의 경계가 존재하는 UDP 소켓
TCP 기반에서 송수신하는 데이터에는 경계가 존재하지 않는데 이는 다음의 의미를 지닌다.

"데이터 송수신 과정에서 호출하는 입출력함수의 호출횟수는 큰 의미를 지니지 않는다."

반대로 UDP 는 데이터의 경계가 존재하는 프로토콜이므로, 데이터 송수신 과정에서 호출하는 입출력함수의 호출횟수가 큰 의미를 지닌다. 때문에 입력함수의 호출횟수와 출력함수의 호출횟수가 완벽히 일치해야 송수신된 데이터 전부를 수신할 수 있다. 예를 들어서 세 번의 출력함수 호출을 통해서 전송된 데이터는 반드시 세 번의 입력함수 호출이 있어야 데이터 전부를 수신할 수 있다.

```c
// host 1
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
  struct sockaddr_in my_adr , your_adr;
  socklen_t adr_sz;
  int str_len , i;

  if(argc != 2) {
    printf("Usage : %s <port> \n" , argv[0]);
    exit(1);
  }

  sock = socket(PF_INET , SOCK_DGRAM , 0);

  if(sock == -1) 
    error_handling("sock() error!");

   memset(&my_adr , 0 , sizeof(my_adr));
  my_adr.sin_family = AF_INET;
  my_adr.sin_addr.s_addr = htonl(INADDR_ANY);
  my_adr.sin_port = htons(atoi(argv[1]));

  if(bind(sock , (struct sockaddr * ) &my_adr , sizeof(my_adr)) == -1) 
    error_handling("bind() error");
  
  for(i = 0; i < 3; i++) {
    sleep(5); // delay

    adr_sz = sizeof(your_adr);
    str_len = recvfrom(sock , message , BUF_SIZE , 0 , (struct sockaddr *) &your_adr , &adr_sz);

    printf("Message : %d : %s \n" , i + 1 , message);
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

```c
// host2
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
  char msg1[] = "HI!";
  char msg2[] = "I'm anther UDP host!";
  char msg3[] = "Nice to meet you";  

  struct sockaddr_in your_adr;
  socklen_t your_adr_sz;

  if(argc != 3) {
    printf("Usage : %s <IP> <port> \n" , argv[0]);
    exit(1);
  }

  sock = socket(PF_INET , SOCK_DGRAM , 0);

  if(sock == -1) 
    error_handling("sock() error!");

  memset(&your_adr , 0 , sizeof(your_adr));
  your_adr.sin_family = AF_INET;
  your_adr.sin_addr.s_addr = inet_addr(argv[1]);
  your_adr.sin_port = htons(atoi(argv[2]));

  sendto(sock , msg1 , sizeof(msg1) , 0 , (struct sockaddr*) &your_adr , sizeof(your_adr));
  sendto(sock , msg2 , sizeof(msg2) , 0 , (struct sockaddr*) &your_adr , sizeof(your_adr));
  sendto(sock , msg3 , sizeof(msg3) , 0 , (struct sockaddr*) &your_adr , sizeof(your_adr));

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
./udp2 8080
Message : 1 : HI! 
Message : 2 : I'm anther UDP host!
Message : 3 : Nice to meet you
```

## connected UDP 소켓,  unconnected UDP 소켓
TCP 소켓에는 데이터를 전송할 목적지의 IP 와 PORT 번호를 등록하는 반면 , UDP 소켓에는 데이터를 전송할 목적지의 IP와 PORT 번호를 등록하지 않는다. 때문에 sendto 함수호출을 통한 데이터의 전송과정은 다음과 같이 크게 세 단계로 나뉜다.

```
1. UDP 소켓에 목적지의 IP와 PORT 번호를 등록
2. 데이터 전송
3. UDP 소켓에 등록된 목적지 정보 삭제
```

즉, sendto 함수가 호출될 때 마다 위의 과정을 반복하게 된다. 이렇듯  목적지의 주소정보가 계속해서 변경되기 때문에 하나의 UDP 소켓을 이용해서 다양한 목적지로 데이터 전송이 가능한 것이다. 그리고 이렇게 목적지 정보가 등록되어 있지 않은 소켓을 가리켜 "unconnected 소켓" 이라고 하고 , 반면 목적지 정보가 등록되어 있는 소켓을 가리켜 "connected 소켓" 이라 한다. 물론 UDP 소켓은 기본적으로 uncconected 소켓이다. 그런데 이러한 UDP 소켓은 다음과 같은 상황에서는 매우 불합리하게 동작한다. 

"IP 211.210.147.82 PORT 82 번으로 준비된 총 세 개의 데이터를 세 번의 sendto 함수호출을 통해서 전송한다.

이 경우 위에서 정리한 데이터 전송 세 단계를 총 3회 반복해야 한다. 그래서 하나의 호스트와 오랜 시간 데이터를 송수신해야한다면, UDP 소켓을 connected 소켓으로 만드는 것이 효율적이다. 참고로 앞서 보인 1단계와 3단계가 UDP 데이터 전송과정의 약 1/3에 해당한다고 하니, 이 시간을 줄임으로 적지 않은 성능향상을 기대할 수 있다.

## connected UDP 소켓 생성
connected UDP 소켓을 생성하는 방법은 이외로 간단한데, UDP 소켓을 대상으로 connect 함수만 호출해주면 된다.

connect 함수를 호출했다고 해서 목적지의 UDP 소켓과 연결설정 과정을 거친다거나 하지는 않는다. 다만 UDP 소켓에 목적지의 IP와 PORT 정보가 등록될 뿐이다. 이로써 이후부터는 TCP 소켓과 마찬가지로 sendto 함수가 호출될 때마다 데이터 전송의 과정만 거치게 된다. 뿐만 아니라 송수신의 대상이 정해졌기 때문에 sendto , recvfrom 함수가 아닌 write , read 함수의 호출로도 데이터를 송수신할 수 ㅣㅇㅆ다.

```c
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

  connect(sock , (struct sockaddr *) &serv_adr , sizeof(serv_adr));

  while(1) {
    fputs("Insert message (q to Quit) : " , stdout);
    fgets(message , sizeof(message) , stdin);
    if(!strcmp(message , "q\n") || !strcmp(message , "Q\n")) 
      break;

    write(sock , message , strlen(message));

    str_len = read(sock , message , sizeof(message));
    message[str_len] = 0;
    printf("Message from server : %s \n" , message);
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