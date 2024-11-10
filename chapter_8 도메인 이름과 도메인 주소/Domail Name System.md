## 도메인 이름이란?
인터넷에서 서비스를 제공하는 서버들 역시 IP 주소로 구분이 된다. 그러나 기억하기 쉽지않은 IP 주소의 형태로 서버의 주소정보를 기억하느 것은 사실상 불가능한 일이다. 대문에 기억하기 좋은 도메인 이름이라는 것을 IP주소에 부여해서 이것이 IP 주소를 대신하도록 하고 있다.

## DNS 서버
인터넷 브라우저 주소 창에 네이버의 IP 주소를 입력하거나 www.naver.com 의 입력을 통해 네이버 접속이 가능하다. 네이버의 메인 페이지에 접속한다는 점에서는 차이가 없지만 접속의 과정에서는 차이가 있다.
이러한 변환을 담당하는 것이 DNS 서버이니, DNS 서버에게 변환을 요청하면 된다.

모든 컴퓨터에는 디폴트 DNS (8.8.8.8) 서버의 주소가 등록되어 있는데 바로 이 디폴트 DNS 서버를 통해서 도메인 이름에 대한 IP 주소 정보를 얻게 된다. 즉, 여러분이 인터넷 브라우저 주소창에 도메인 이름을 입력하면 인터넷 브라우저는 해당 도메인 이름의 IP 주소를 디폴트 DNS 서버를 통해 얻게 되고 그 다음에 비로소 서버로의 실제 접속에 들어가게 된다.

디폴트 DNS 서버는 모르면 다른 DNS 서버에게 물어본다.

디폴트 DNS 서버는 자신이 모르는 정보에 대한 요청이 들어오면 한 단계 상위 계층에 있는 DNS 서버에게 물어본다. 이러한 식으로 계속 올라가다 보면 최상위 DNS 서버인 Root DNS 서버에게까지 질의가 전달되는데, Root DNS 서버는 해당 질문을 누구에게 재 전달해야 할지 알고 있다. 그래서 자신보다 하위에 있는 DNS 서버에게 다시 질의를 던져서 결국 IP 주소를 얻어낸다. 그 결과는 질의가 진행된 반대방향으로 전달되어 호스트에게 IP 주소가 전달된다. 이렇듯 DNS 는 계층적으로 관리되는 일종의 분산 데이터베이스 시스템이다.

## 도메인 이름을 이용해서 IP 주소 얻어오기
다음 함수를 이용하면 문자열 형태의 도메인이름으로부터 IP 의 주소정보를 얻을 수 있다.

```c
#include <netdb.h>

struct hostent * gethostbyname(const char * hostname); // 성공 시 hostent 구조체 변수의 주소 값 , 실패 시 NULL 포인터 반환
```

hostent 라는 구조체의 변수에 담겨서 반환이 되는데 이 구조체는 다음과 같이 정의되었다.

```c
struct hostent {
  char * h_name; // official name
  char ** h_aliases; // alias list
  int h_addrtype; // host address type
  int h_length; // address length
  char ** h_addr_list; // address list
}
```

IP 정보만 반환되는 것이 아니라 여러가지 다른 정보들도 덤으로 반환되는 것을 알 수 있다. 도메인 이름을 IP 로 변환하는 경우에는 h_addr_list 만 신경써도 된다. 간단히 멤버 각각에 대해 알아보자

### h_name
이 멤버에는 '공식 도메인 이름(Official domain name)' 이라는 것이 저장된다.

### h_aliases
같은 메인 페이지인데도 다른 도메인 이름으로 접속할 수 있는 경우를 본적 있는가? 하나의 IP 에 둘 이상의 도메인 이름을 지정하는 것이 가능하기 때문에 공식 도메인 이름 이외에 해당 메인 페이지에 접속할 수 있는 다른 도메인 이름의 지정이 가능하다. 그리고 이들 정보는 h_aliases 를 통해서 얻을 수 있다.

### h_addrtype
gethostbyname 함수는 IPv4 뿐만 아니라 IPv6까지 지원한다. 때문에 h_addr_list 로 반환된 IP 주소의 주소체계에 대한 정보를 이 멤버를 통해 반환한다. IPv4의 경우 이 멤버에는 AF_INET 이 저장된다.

### h_length
함수호출의 결과로 반환된 IP주소의 크기 정보가 담긴다. IPv4 의 경우 4바이트이므로 4가 저장되고 IPv6의 경우에 16바이트 이므로 16이 저장된다

### h_addr_list
이것이 가장 중요한 멤버이다. 이 멤버를 통해서 도메인 이름에 대한 IP 주소가 정수의 형태로 반환된다. 참고로 접속자수가 많은 서버는 하나의 도메인 이름에 대응하는 IP를 여러 개 둬서, 둘 이상의 서버로 부하를 분산시킬 수 있는데, 이러한 경우에도 이 멤버를 통해서 모든 IP의 주소정보를 얻을 수 있다.

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <netdb.h>

void error_handling(char * message);


int main(int argc, char const *argv[])
{
  int i;
  struct hostent *host;
  
    if(argc != 2) {
    printf("Usage : %s <addr> \n" , argv[0]);
    exit(1);
  }

  host = gethostbyname(argv[1]);

  if(!host) 
    error_handling("gethost ... error");

  // 공식 도메인 이름
  printf("official name : %s \n" , host->h_name);

  // aliases 이름
  for(i = 0; host -> h_aliases[i]; i++) 
    printf("Aliases %d : %s \n" , i + 1 , host ->h_aliases[i]);
  
  // address type
  printf("Address type : %s \n" , host -> h_addrtype == AF_INET ? "AF_INET" : "AF_INET6");

  // ip addr
  for(i = 0 ; host -> h_addr_list[i]; i++)
    printf("IP addr %d : %s \n" , i + 1 , inet_ntoa(* (struct in_addr *) host -> h_addr_list[i]));

  return 0;
}

void error_handling(char * message) {
  fputs(message , stderr);
  fputc('\n' , stderr);
  exit(1);
}
```

```
$ ./gethostbyname www.naver.com

official name : www.naver.com.nheos.com 
Aliases 1 : www.naver.com 
Address type : AF_INET 
IP addr 1 : 223.130.192.247 
IP addr 2 : 223.130.192.248 
IP addr 3 : 223.130.200.219 
IP addr 4 : 223.130.200.236 
```

구조체 멤버 h_addr_list 가 가리키는 것은 문자열 포인터 배열 (둘 이상의 문자열 주소 값로 구성된 배열)이다. 그러나 문자열 포인터 배열이 실제 가리키는 것은 (실제 저장하는 것은)문자열의 주소값이 아닌 in_addr 구조체 변수의 주소 값이다.

## IP 주소를 이용해서 도메인 정보 얻어오기
앞서 소개한 gethostbyname 함수는 도메인 이름을 이용해서 IP 주소를 포함한 도메인 정보를 얻을 때 호출하는 함수이다. 반면 이번에 소개하는 gethostbyaddr 함수는 IP 주소를 이용해서 도메인 정보를 얻을 때 호출하는 함수이다.

```c
#include <netdb.h>

struct hostent * gethostbyaddr(const char * addr , socklen_t len , int family); // 성공 시 hostent 구조체 변수의 주소 값 , 실패 시 NULL 포인터 반환

// addr : IP 주소를 지니는 in_addr 구조체 변수의 포인터 전달 IPv4 이외의 다양한 정보를 전달받을 수 있도록 일반화하기 위해서 매개변수를 char 포인터로 선언
// len : 첫 번째 인자로 전달된 주소정보의 길이 IPv4 의 경우 4 , IPv6 의 경우 16 전달
// family : 주소체계 정보 전달. IPv4 의 경우 AF_INET , IPv6의 경우 AF_INET6 전달
```

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <netdb.h>

void error_handling(char * message);


int main(int argc, char const *argv[])
{
  int i;
  struct hostent *host;
  struct sockaddr_in addr;

    if(argc != 2) {
    printf("Usage : %s <IP> \n" , argv[0]);
    exit(1);
  }

  memset(&addr , 0 , sizeof(addr));
  addr.sin_addr.s_addr = inet_addr(argv[1]);

  host = gethostbyaddr((char*) &addr.sin_addr , 4 , AF_INET);

  if(!host) 
    error_handling("gethost ... error");

  // 공식 도메인 이름
  printf("official name : %s \n" , host->h_name);

  // aliases 이름
  for(i = 0; host -> h_aliases[i]; i++) 
    printf("Aliases %d : %s \n" , i + 1 , host ->h_aliases[i]);
  
  // address type
  printf("Address type : %s \n" , host -> h_addrtype == AF_INET ? "AF_INET" : "AF_INET6");

  // ip addr
  for(i = 0 ; host -> h_addr_list[i]; i++)
    printf("IP addr %d : %s \n" , i + 1 , inet_ntoa(* (struct in_addr *) host -> h_addr_list[i]));

  return 0;
}

void error_handling(char * message) {
  fputs(message , stderr);
  fputc('\n' , stderr);
  exit(1);
}
```

```
[root@localhost chapter5]# ./gethostbyname www.google.co.kr
official name : www.google.co.kr 
Address type : AF_INET 
IP addr 1 : 172.217.161.35 

[root@localhost chapter5]# ./gethostbyaddr 172.217.161.35
official name : nrt12s23-in-f3.1e100.net 
Address type : AF_INET 
IP addr 1 : 172.217.161.35 
```

