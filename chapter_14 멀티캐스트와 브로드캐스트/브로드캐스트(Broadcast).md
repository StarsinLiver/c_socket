## 브로드캐스트 (Broadcast)
브로드캐스트는 한번에 여러 호스트에게 데이터를 전송한다는 점에서 멀티캐스트와 유사하다. 그러나 전송이 이뤄지는 범위에서 차이가 난다. 멀티캐스트는 서로 다른 네트워크상에 존재하는 호스트라 할지라도 멀티캐스트 그룹에 가입만 되어 있으면 데이터의 수신이 가능하다. 반면 브로드캐스트는 동일한 네트워크로 연결되어 있는 호스트로 데이터의 전송 대상이 제한된다.

## 브로드캐스트의 이해와 구현방법
브로드캐스트는 동일한 네트워크에 연결되어있는 모든 호스트에게 동시에 데이터를 전송하기 위한 방법이다. 이 역시 멀티캐스트와 마찬가지로 UDP를 기반으로 데이터를 송수신한다. 그리고 데이터 전송시 사용되는 IP주소의 형태에 따라서 다음과 같이 두 가지 형태로 구분이 된다.

```
1. Directed 브로드캐스트
2. Local 브로드캐스트
```

코드상에서 확인되는 이 둘의 차이점은 IP주소에 있다. Directed 브로드캐스트의 IP 주소는 네트워크 주소를 제외한 나머지 호스트 주소를 전부 1로 설정해서 얻을 수 있다. 예를 들어서 네트워크 주소가 192.12.34 인 네트워크에 연결되어 있는 모든 호스트에게 데이터를 전송하려면 192.12.34.255로 데이터를 전송하면 된다. 이렇듯 특정 지역의 네트워크에 연결된 모든 호스트에게 데이터를 전송하려면 Directed 브로드캐스트 방식으로 데이터를 전송하면 된다.

반면, Local브로드캐스트를 위해서는 255.255.255.255 라는 IP주소가 특별히 예약되어 있다. 예를 들어서 네트워크 주소가 192.32.24 인 네트워크에  연결되어있는 호스트가 IP주소 255.255.255.255 를 대상으로 데이터를 전송하면 , 192.32.24로 시작하는 IP 주소의 모든 호스트에게 데이터가 전달된다.

기본적으로 생성되는 소켓은 브로드캐스트 기반의 데이터 전송이 불가능하도록 설정되어 있기때문에 다음과 같이 변경을 할 필요가 있다.

```
int send_sock;
int bcast = 1; // SO_BROADCAST 의 옵션정보를 1로 변경하기 위한 변수 초기화
send_sock = socket(PF_INET, SOCK_DGRAM , 0);
setsockopt(send_+sock , SOL_SOCKET , SO_BROADCAST , (void*) &bcast , sizeof(bcast));
```

위의 setsockopt 함수호출을 통해서 SO_BROADCAST 의 옵션정보를 변수 bcast에 저장된 값인 1로 변경하는데 이는 브로드캐스트 기반의 데이터 전송이 가능함을 의미한다. 물론 위에서 보인 소켓옵션의 변경은 데이터를 전송하는 Sender 에게나 필요할 뿐 Receiver의 구현에서는 필요없다.

## 브로드캐스트 기반의 Sender 와 Receiver 의 구현
이제 브로드캐스트 기반의 Sender 와 Receiver 를 구현해 보자

```c
// news_sender_brd.c
#define _GNU_SOURCE // gnu 확장기능
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h> // ip_mreq 파일

#define BUF_SIZE 30
void error_handling(char * message);

int main(int argc, char const *argv[])
{
  int send_sock;
  struct sockaddr_in broad_adr;
  FILE *fp;
  char buf[BUF_SIZE];
  int so_brd = 1;

  if(argc != 3) {
    printf("Usage : %s <GruopIP> <port> \n" , argv[0]);
  }

  send_sock = socket(PF_INET , SOCK_DGRAM , 0);
  memset(&broad_adr , 0 , sizeof(broad_adr));
  broad_adr.sin_family = AF_INET;
  broad_adr.sin_addr.s_addr = inet_addr(argv[1]); // Multicast IP
  broad_adr.sin_port = htons(atoi(argv[2])); // Multicast Port

  setsockopt(send_sock , SOL_SOCKET , SO_BROADCAST , (void*) &so_brd , sizeof(so_brd));
  
  if((fp = fopen("news.txt" ,"r")) == NULL) {
    error_handling("fopen() error!");
  }

  while(!feof(fp)) { /* Broadcasting */
    fgets(buf , BUF_SIZE , fp);
    sendto(send_sock , buf , strlen(buf) , 0 , (struct sockaddr*)&broad_adr , sizeof(broad_adr));

    sleep(2);
  }

  close(fp);
  close(send_sock);
  return 0;
}

void error_handling(char * message) {
  fputs(message , stderr);
  fputc('\n' , stderr);
  exit(1);
}
```

```c
// news_receiver_brd.c
#define _GNU_SOURCE // gnu 확장기능
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h> // ip_mreq 파일

#define BUF_SIZE 30
void error_handling(char * message);

int main(int argc, char const *argv[])
{
  int recv_sock;
  int str_len;
  char buf[BUF_SIZE];
  struct sockaddr_in adr;

  if(argc != 2) {
    printf("Usage : %s <GroupIP> <port>\n", argv[0]);
    exit(1);  // 프로그램 종료
  }

  recv_sock = socket(PF_INET, SOCK_DGRAM, 0);
  if (recv_sock == -1)
    error_handling("socket() error");

  memset(&adr, 0, sizeof(adr));
  adr.sin_family = AF_INET;
  adr.sin_addr.s_addr = htonl(INADDR_ANY);  // 수정된 부분
  adr.sin_port = htons(atoi(argv[1]));

  if(bind(recv_sock, (struct sockaddr*) &adr, sizeof(adr)) == -1)
    error_handling("bind() error!");

  while(1) {
    str_len = recvfrom(recv_sock, buf, BUF_SIZE - 1, 0, NULL, 0);
    if(str_len < 0)
      break;
    buf[str_len] = 0;
    fputs(buf, stdout);
  }

  close(recv_sock);
  return 0;
}

void error_handling(char * message) {
  fputs(message, stderr);
  fputc('\n', stderr);
  exit(1);
}
```

```
./news_sender_brd 255.255.255.255 9190
./news_receiver_brd 9190
```