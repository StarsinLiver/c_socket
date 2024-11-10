## Urgent mode 에 대해 좀 더 보충설명
"하이~ 긴급으로 처리해야할 데이터가 들어갔으니 꾸물거리지 마!"

즉, 데이터를 수신하는 대상에게 데이터의 처리를 독촉하는데 MSG_OOB의 진정한 의미가 있다. 이것이 전부이고 데이터의 전송에는 "전송순서가 그대로 유지된다" 라는 TCP의 전송 특성은 그대로 유지된다.

그럼 간단히 MSG_OOB 옵션이 설정된 상태에서의 데이터 전송과정을 간단히 설명해보자

![alt text](/image/21.png)

위의 그림은 예제 oob_send.c의  32행 에서의 다음 함수 호출 후의 출력버퍼 상황을 보인다. 물론 이전의 데이터는 이미 전송되었다는 가정이 존재한다.

```
send(sock , "89-) , strlen("890" , MSG_OOB);
```

버퍼의 가장 왼쪽 위치를 오프셋 0으로 보면, 문자 0은 오프셋 2의 위치에 저장되어있는 상황이다. 그리고 문자 0의 오른편인 오프셋 3의 위치가 Urgent Pointer 로 지정되었다고 말할 수 있다. Urgent Pointer는 긴급 메시지의 다음 번(오프셋이 1 증가한) 위치를 가리키면서 다음의 정보를 상대 호스트에게 전달하는 의미를 가진다.

"Urgent Pointer가 가리키는 오프셋 3의 바로 앞에 존재하는 것이 긴급메시지야!"

실제로 TCP 패킷에는 보다 많은 정보가 들어가지만 현재 우리가 하는 이야기에 관련있는 내용만 위 그림에다 표시했다.

```
URG=1         : 긴급 메시지가 존재하는 패킷이다.
URG POINTER   : Urgent Pointer의 위치가 오프셋 3의 위치에 있다.
```

즉, MSG_OOB옵션이 지정되면 패킷 자체가 긴급 패킷이 되며, Urgent Pointer를 통해서 긴급 메시지의 위치도 표시된다.

"긴급 메시지가 문자열 890이야, 아니면 90이야! 그것도 아니면 그냥 문자 0 하나인가?"

그런데 이는 별로 중요하지 않다. 앞서 예제에서 확인했듯이 어차피 이 데이터를 수신하는 상대방은 Urgent Pointer의 앞부부분에 위치한 1바이트를 제외한 나머지는 일반적인 입력함수의 호출을 통해서 읽히기 때문이다.
즉, 긴급 메시지는 메시지 처리를 재촉하는데 의미가 있는 것이지 제한된 형태의 메시지를 긴급으로 전송한느데 의미가 있는 것은 아니다.

offset은 기준점으로 부터 어느 쪽으로 얼마나 떨어져 있는지를 나타내는 도구가 된다. 때문에 일반적인 주소와 달리 항상 0에서 부터 시작한다.

## 입력버퍼 검사하기
MSG_PEEK 옵션은 MSG_DONTWAIT옵션과 함께 설정되어 입력버퍼에 수신된 데이터가 존재하는 지 확인하는 용도로 사용된다. MSG_PEEK 옵션을 주고 recv함수를 호출하면 입력버퍼에 존재하는 데이터가 읽혀지더라도 입력버퍼에서 데이터가 지워지지 않는다. 때문에 MSG_DONTWAIT 옵션과 묶어 블로킹되지 않는 데이터의 존재유무를 확인하기 위한 함수의 호출구성에 사용된다.

```c
// peek_recv.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#define BUF_SIZE 30
void error_handling(char * message);

int main(int argc, char const *argv[])
{

  int acpt_sock , recv_sock;
  struct sockaddr_in acpt_adr ,recv_adr;
  int str_len , state;
  socklen_t recv_adr_sz;
  char buf[BUF_SIZE];

  if(argc != 2) {
    printf("Usage : %s <port> \n" , argv[0]);
  }

  acpt_sock = socket(PF_INET , SOCK_STREAM , 0);
  memset(&acpt_adr , 0 , sizeof(acpt_adr));
  acpt_adr.sin_family = AF_INET;
  acpt_adr.sin_addr.s_addr = htonl(INADDR_ANY);
  acpt_adr.sin_port = htons(atoi(argv[1]));

  if(bind(acpt_sock , (struct sockaddr*) &acpt_adr , sizeof(acpt_adr)) == -1)
    error_handling("bind() error!");

  listen(acpt_sock , 5);

  recv_adr_sz = sizeof(recv_adr);
  recv_sock = accept(acpt_sock , (struct sockaddr*) &recv_adr , &recv_adr_sz);

  while(1) {
    str_len = recv(recv_sock , buf , sizeof(buf) - 1 , MSG_PEEK | MSG_DONTWAIT);
    if(str_len > 0) 
      break;
  }

  buf[str_len] = 0;
  printf("Buffering %d bytes : %s \n" , str_len , buf);

  str_len = recv(recv_sock , buf , sizeof(buf) - 1 , 0);
  buf[str_len] = 0;
  printf("Read again %s \n" , buf);
  close(acpt_sock);
  close(recv_sock);
  return 0;
}


void error_handling(char * message) {
  fputs(message , stderr);
  fputc('\n' , stderr);
  exit(1);
}
```

```c
// peek_send.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>

void error_handling(char * message);

int main(int argc, char const *argv[])
{
  int sock;
  struct sockaddr_in send_adr;

  if(argc != 3) {
    printf("Usage : %s <IP> <port> \n" , argv[0]);
  }

  sock = socket(PF_INET , SOCK_STREAM , 0);
  memset(&send_adr , 0 , sizeof(send_adr));
  send_adr.sin_family = AF_INET;
  send_adr.sin_addr.s_addr = inet_addr(argv[1]);
  send_adr.sin_port = htons(atoi(argv[2]));

  if(connect(sock , (struct sockaddr*) &send_adr , sizeof(send_adr)) == -1) 
    error_handling("connect() error!");

  write(sock , "123" , strlen("123"));
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
[root@localhost chapter13]# gcc peek_recv.c -o peek_recv
[root@localhost chapter13]# ./peek_recv 9190
[root@localhost chapter13]# ./peek_send 127.0.0.1 9190
Buffering 3 bytes : 123 
Read again 123 
```