## 멀티캐스트 (Multicast)
멀티캐스트 방식의 데이터 전송은 UDP 를 기반으로 한다. 따라서 UDP 서버/클라이언트의 구현방식이 매우 유사하다. 차이점이 있다면 UDP에서의 데이터 전송은 하나의 목적지를 두고 이뤄지나 멀티캐스트에서의 데이터 전송은 특정 그룹에 가입(등록) [IGMP] 되어 있는 다수의 호스트가 된다는 점이다. 즉, 멀티캐스트 방식을 이용하면 단 한번에 데이터 전송으로 다수의 호스트에게 데이터를 전송이 가능하다.

## 멀티캐스트의 데이터 전송방식과 멀티캐스트 트래픽 이점
멀티캐스트의 데이터 전송특성은 다음과 같이 간단히 정리할 수 있다.

```
1. 멀티캐스트 서버는 특정 멀티캐스트 그룹을 대상으로 데이터를 딱 한번 전송한다.
2. 딱! 한번 전송하더라도 그룹에 속하는 클라이언트는 모두 데이터를 수신한다.
3. 멀티캐스트 그룹의 수는 IP 주소 범위 내에서 얼마든지 추가가 가능하다.
4. 특정 멀티캐스트 그룹으로 전송되는 데이터를 수신하려면 해당 그룹에 가입하면 된다.
```

여기서 말하는 멀티캐스트 그룹이란 클래스 D에 속하는 IP 주소(224.0.0.0 ~ 239.255.255.255)를 조금 폼나게 표현한 것에 지나지 않는다. 따라서 멀티캐스트 그룹에 가입한다는 것은 프로그램 코드상에서 다음과 같이 외치는 것 정도로 이해할 수 있다.

"나는 클래스 D에 속하는 IP 주소 중에서 239.234.218.234를 목적지로 전송되는 멀티캐스트 데이터에 관심이 있으므로 이 데이터를 수신하겠다."

멀티캐스트는 UDP 를 기반으로 한다고 하였다. 즉, 멀티캐스트 패킷은 그 형태가 UDP 패킷과 동일하다. 다만 일반적인 UDP 패킷과 달리 하나의 패킷만 네트워크상에 띄워 놓으면 라우터들은 이 패킷을 복사해서 다수의 호스트에 이를 전달한다.이렇듯 멀티캐스트는 다음 그림에서 보이듯이 라우터의 도움으로 완성된다.

![alt text](/image/24.png)

(그림은 다른데..) 위 그림은 그룹 AAA로 전송된 하나의 멀티캐스트 패킷이 라우터들의 도움으로 AAA그룹에 가입한 모든 호스트에 전송되는 과정을 보이고 있다. 

"이렇듯 다수의 클라이언트에게 동일한 데이터를 전송하는 일 조차 서버와 네트워크의 트래픽 측면에서 매우 부정적이다. 그러나 이러한 상황에서의 해결책으로 멀티캐스트라는 기술이 존재한다."

단순히 보면 트래픽에 부정적이라 생각할 수 있다. 왜냐하면 하나의 패킷이 여러 라우터를 통해서 빈번히 복사되기 때문이다. 하지만 다음 측면으로 관찰하자!

"하나의 영역에 동일한 패킷이 둘 이상 전송되지 않는다."

만약에 TCP또는 UDP 방식으로 1000개의 호스트에 파일을 전송하려면 총 1000회 파일을 전송해야한다. 열개의 호스트가 하나의 네트워크로 묶여 있어서 경로의 99%가 일치하더라도 말이다. 하지만 이러한 경우에 멀티캐스트 방식으로 파일을 전송하면 딱 한번만 전송해주면 된다. 1000개의 호스트를 묶고 있는 라우터가 1000개의 호스트에게 파일을 복사해줄테니 말이다. 바로 이러한 성격때문에 멀티캐스트 방식의 데이터 전송은 "멀티미디어 데이터의 실시간 전송"에 주로 사용된다.

다만 이론상으로는 쉽게 가능하나 아직 적지 않은 수의 라우터가 멀티캐스트를 지원하지 않거나 지원하더라도 네트워크의 불필요한 트래픽 문제를 고려해서 일부러 막은 경우가 많다. 때문에 멀티캐스트를 지원하지 않는 라우터를 거쳐서 멀태캐스트 패킷을 전송하기 위한 터널링(Tunneling) 기술이라는 것도 사용이된다.

어찌됐는 우리는 멀티캐스트 서비스가 가능한 환경이 구축되어 있는 상황에서의 프로그래밍 방법에 대해서만 이야기 해보자

## 라우팅(Routing) 과 TTL(Time to Live) 그리고 그룹으로의 가입 방법 [Chapter 09 소켓옵션 참조]
그럼 이제 멀티캐스트 관련 프로그래밍에 대해 이야기 해보자

TTL 이란 Time to Live 의 약자로써 "패킷을 얼마나 멀리 전달할 것인가"를 결정하는 주 요소가 된다. TTL은 정수로 표현되며 이 값은 라우터를 하나 거칠때마다 1씩 감소하며 이 값이 0이 되면 패킷은 더 이상 전달되지 못하고 소멸된다.

그럼 TTL 의 설정방법에 대해 설명하겠다. 프로그램상에서의 TTL 설정은 소켓의 옵션설정을 통해 이뤄진다. TTL 의 설정과 관련된 프로토콜의 레벨은 IPPROTO_IP이고 옵션의 이름은 IP_MULTICAST_TTL이다. 따라서 TTL을 64로 설정하고자 할 때에는 다음과 같이 구성하면 된다.

```c
int send_sock;
int time_live = 64;

send_sock = socket(PF_INET , SOCK_STREAM , 0);
setsockopt(send_sock , IPPROTO_IP , IP_MULTICAST_TTL , (void*) &time_live , sizeof(time_live));
```

그리고 멀티캐스트 그룹으로의 가입 역시 소켓의 옵션설정을 통해 이루어 진다. 그룹 가입과 관련된 프로토콜의 레벨은 IPPROTO_IP이고 옵션의 이름은 IP_ADD_MEMBERSHIP이다. 따라서 그룹의 가입은 다음과 같이 진행된다.

```c
int recv_sock;
strucvt ip_mreq join_adr;

recv_sock = socket(PF_INET , SOCK_STREAM , 0);

join_adr.imr_multiaddr.s_addr="멀티캐스트 그룹의 주소정보";
join_adr.imr_interface.s_addr="그룹에 가입할 호스트의 주소정보";

setsockopt(recv_sock , IPPPROTO_IP , IP_ADD_MEMBERSHIP , (void*)&join_adr , sizeof(join_adr));
```

위의 코드는 setsockopt 함수의 호출과 관련있는 부분만 보였을 뿐이다. 자세한것은 잠시 후에 소개하겠다.

구조체 ip_mreq 는 다음과 같다.

```c
#define _GNU_SOURCE // gnu 확장기능
#include <netinet/in.h> // ip_mreq 파일
struct ip_mreq {
  struct in_addr imr_multiaddr;
  struct in_addr imr_interface;
}
```

구조체 in_addr 은 chapter 03 을 참조하자 

우선 첫번째 맴버 imr_multiaddr에는 가입할 그룹의 IP주소를 채워 넣는다. 그리고 두 번째 맴버인 imr_interface 에는 그룹에 가입하는 소켓이 속한 호스트의 IP 주소를 명시하는데 INADDR_ANY를 이용하는 것도 가능하다.

## 멀티캐스트 Sender 와 Receiver 구현
멀티캐스트 기반에서는 서버, 클라이언트 표현을 대신하서 전송자 (이하 Sender) 와 수신사 (이하 Receiver)라는 표현을 사용하자 여기서 Sender 는 말 그대로 멀티캐스트 데이터의 전송주체이다. 반면 Receiver는 멀티캐스트 그룹의 가입과정이 필요한 데이터의 수신주체이다. 그럼 이어서 구현할 예제에 대해 이야기 해 보자

```
Sender : 파일에 저장된 뉴스 정보를 AAA 그룹으로 방송(Broadcasting)한다.
Receiver : AAA 그룹으로 전송된 뉴스정보를 수신한다.
```

이제 Sender 의 코드를 볼 텐데 Sender 는 Receiver 보다 상대적으로 간단하다. Receiver는 그룹의 가입 과정이 필요하지만, Sender 는 UDP 소켓을 생성하고 멀티캐스트 주소로 데이터를 전송만 하면 되기 때문이다.


```c
// Sender 
#define _GNU_SOURCE // gnu 확장기능
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h> // ip_mreq 파일

#define TTL 64
#define BUF_SIZE 30
void error_handling(char * message);

int main(int argc, char const *argv[])
{
  int send_sock;
  struct sockaddr_in mul_adr;
  int time_live = TTL;
  FILE *fp;
  char buf[BUF_SIZE];

  if(argc != 3) {
    printf("Usage : %s <GruopIP> <port> \n" , argv[0]);
  }

  send_sock = socket(PF_INET , SOCK_DGRAM , 0);
  memset(&mul_adr , 0 , sizeof(mul_adr));
  mul_adr.sin_family = AF_INET;
  mul_adr.sin_addr.s_addr = inet_addr(argv[1]); // Multicast IP
  mul_adr.sin_port = htons(atoi(argv[2])); // Multicast Port

  setsockopt(send_sock , IPPROTO_IP , IP_MULTICAST_TTL , (void*) &time_live , sizeof(time_live));
  
  if((fp = fopen("news.txt" ,"r")) == NULL) {
    error_handling("fopen() error!");
  }

  while(!feof(fp)) { /* Broadcasting */
    fgets(buf , BUF_SIZE , fp);
    sendto(send_sock , buf , strlen(buf) , 0 , (struct sockaddr*)&mul_adr , sizeof(mul_adr));

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
// Receiver
#define _GNU_SOURCE // gnu 확장기능
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <netinet/in.h> // ip_mreq 파일

#define TTL 64
#define BUF_SIZE 30
void error_handling(char * message);

int main(int argc, char const *argv[])
{
  int recv_sock;
  int str_len;
  char buf[BUF_SIZE];
  struct sockaddr_in adr;
  struct ip_mreq join_adr;

  if(argc != 3) {
    printf("Usage : %s <GroupIP> <port>\n", argv[0]);
    exit(1);  // 프로그램 종료
  }

  recv_sock = socket(PF_INET, SOCK_DGRAM, 0);
  if (recv_sock == -1)
    error_handling("socket() error");

  memset(&adr, 0, sizeof(adr));
  adr.sin_family = AF_INET;
  adr.sin_addr.s_addr = htonl(INADDR_ANY);  // 수정된 부분
  adr.sin_port = htons(atoi(argv[2]));

  if(bind(recv_sock, (struct sockaddr*) &adr, sizeof(adr)) == -1)
    error_handling("bind() error!");

  join_adr.imr_multiaddr.s_addr = inet_addr(argv[1]);
  join_adr.imr_interface.s_addr = htonl(INADDR_ANY);

  if (setsockopt(recv_sock, IPPROTO_IP, IP_ADD_MEMBERSHIP, (void*) &join_adr, sizeof(join_adr)) == -1)
    error_handling("setsockopt() error!");  // 오류 처리 추가

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
[root@localhost chapter14]# ./news_sender 224.1.1.2 9190
[root@localhost chapter14]# ./news_receiver 224.1.1.2 9190

(잘 안된다...)
```

## MBone (Multicast Backbone)
멀티캐스트는 MBone 이라 불리는 가상 네트워크를 기반으로 동작한다. 일단 가상 네트워크라는 표현이 생소하게 들리는데 이는 인터넷상에서 별도의 프로토콜을 기반으로 동작하는, 소프트웨어적인 개념의 네트워크로 이해할 수 있다. 즉, MBone 은 손으로 만질 수 있는 물리적인 개념의 네트워크가 아니다. 멀티캐스트에 필요한 네트워크 구조를 인터넷 망을 바탕으로 소프트웨어적으로 구현해 놓은 가상의 네트워크이다. 멀티캐스트가 가능하도록 돕는 가상 네트워크의 연구는 지금도 계속되고 있으며, 이는 멀티캐스트 기반의 응용 소프트웨어를 개발하는 것과는 다른 분야의 연구이다.