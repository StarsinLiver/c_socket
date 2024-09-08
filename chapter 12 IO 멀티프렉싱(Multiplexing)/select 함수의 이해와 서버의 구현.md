## select 함수의 이해와 서버의 구현
select 함수를 이용하는 것이 멀티플렉싱 서버의 구현에 있어 가장 대표적인 방법이다.

## select 함수의 기능과 호출순서
select 함수를 사용하면 한곳에 여러 개의 파일 디스크립터를 모아놓고 동시에 이들을 관찰할 수 있다. 이때 관찰할 수 있는 항목은 다음과 같다.

```
1. 수신한 데이터를 지니고 있는 소켓이 존재하는가?
2. 블로킹되지 않고 데이터의 전송이 가능한 소켓은 무엇인가?
3. 예외상황이 발생한 소켓은 무엇이가?
```

위에서 정리한 관찰항목 각각을 가리켜 이벤트라고 하고, 관찰항목에 속하는 상황이 발생했을 때, '이벤트(event가 발생했다)'라고 표현한다.

그런데 select 함수는 사용방법에 있으서 일반적인 함수들과 많은 차이를 보인다. 이제 select 함수의 호출방법과 순서를 알아보자

![alt text](/image/17.png)

위 그림은 select 함수를 호출해서 결과를 얻기까지의 과정을 간략히 정리한 것이다.

## 파일 디스크립터의 설정
select 함수를 사용하면 여러개의 파일 디스크립터를 동시에 관찰할 수 있다고 하였다. 물론 파일 디스크립터의 관찰은 소켓의 관찰로 해석할 수 있다. 그렇다면 먼저 관찰하고자 하는 파일 디스크립터를 모아야한다. 모을 대도 관찰항목(수신 , 전송 , 예외)에 따라서 구분해서 모아야 한다. 즉, 바로 위에서 언급한 세가지 관찬 항목별로 구분해서 세 묶음을 모아야 한다.

파일 디스크립터를 세 묶음으로 모을 때 사용되는 것이 fd_set 형 변수이다. 이는 0 과 1로 표현되는, 비트 단위로 이뤄진 배열이라고 생각하면 된다.

![alt text](/image/19.png)

배열에서 가장 왼쪽 비트는 파일 디스크립터 0을 나타낸다 이 비트가 1로 설정되면 해당 파일 디스크립터가 관찰의 대상임을 의미한다. 위의 그림에서는 fd1 과 fd3 가 관찰대상인 것이다.

"그럼 파일 디스크립터의 숫자를 확인해서 fd_set 형 변수에 직접 값을 등록해야 하는가?"

아니다! fd_set 형 변수의 조작은 비트단위로 이뤄지기 때문에 직접 값을 등록하거나 변경하는 등의 작업은 다음 매크로 함수들의 도움을 통한다.

```
FD_ZERO(fd_set * fdset) : 인자로 전달된 주소의 fd_set 형 변수의 모든 비트를 0으로 초기화
FD_SET(int fd , fd_set *fdset) : 매개변수 fdset으로 전달된 주소의 변수에 매개변수 fd 로 전달된 파일 디스크립터 정보 등록
FD_CLR(int fd , fd_set *fdset) : 매개변수 fdset 으로 전달된 주소의 변수에서 매개변수 fd 로 전달된 파일 디스크립터 정보 삭제
FD_ISSET(int fd , fd_set * fdset) : 매개변수 fdset으로 전달된 주소의 변수에 매개변수 fd로 전달된 파일 디스크립터 정보가 있으면 양수를 반환
```

위의 함수중 FD_ISSET 은 select ㅎ마수의 호출 결과를 확인하는 용도로 사용된다.

![alt text](/image/20.png)

## 검사(관찰)의 범위 지정과 타임아웃 설정
'파일 디스크립터 설정'이외의 나머지 두 가지를 설명하겠다. 그 전에 select 함수를 먼저 보이고 설명하겠다.

```c
#include <sys/select.h>
#include <sys/time.h>

int select (int maxfd , fd_set *readset , fd_set *writeset , fd_set *exceptset , const struct timeval * timeout); // 성공 시 0 , 실패시 -1

// maxfd : 검사 대상이 되는 파일 디스크립터의 수
// readset : fd_set형 변수에 '수신된 데이터의 존재여부'에 관심있는 파일 디스크립터 정보를 모두 등록해서 그 변수의 주소 값을 전달
// writeset : fd_st 형 변수에 '블로킹 없는 데이터 전송의 가능여부'에 관심있는 파일 디스크립터 정보를 모두 등록해서 그 변수의 주소 값 전달
// exceptset : fd_set 형 변수에 '예외 상황의 발생여부'에 관심이 있는 파일 디스크립터 정보를 모두 등록해서 그 변수의 주소 값을 전달
// timeout : select 함수호출 이후에 무한정 블로킹 상태에 빠지지 않도록 타임아웃(time-out)을 설정하기 위한 인자 전달

// 반환 값 : 오류 발생시 -1 , 타임아웃에 의한 반환 시 0 , 그리고 관심대상으로 등록된 파일 디스크립터에 해당 관심에 관련된 변화가 발생시 0보다 큰값 (이 값은 변화가 발생한 파일 디스크립터의 수를 의미)
```

select 함수는 세가지 관찰항목의 변화를 확인하는 데 사용된다고 하지 않았는가? 바로 이 세가지 관찰항목별로 fd_set형 변수를 선언해서 파일 디스크립터 정보를 등록하고, 이 변수의 주소 값을 위 함수의 두 , 세번째 그리고 네번째 인자로 전달하게 된다.

첫번째, 파일 디스크립터의 관찰(검사) 범위는 select 함수의 첫번째 매개변수와 관련된다. fd_set 형 변수에 등록된 파일 디스크립터의 수를 확인할 필요가 있는데 파일 디스크립터의 값은 생성될 때마다 1씩 증가하기 때문에 가장 큰 파일 디스크립터의 값에 1을 더해서 인자로 전달하면 된다. 1을 더하는 이유는 파일 디스크립터의 값이 0에서부터 시작하기 때문이다.

두번째 , select 함수의 타임아웃 시간은 select 함수의 마지막 매개변수와 관련이 있다. 매개변수 선언에서 보이는 자료형 timeval 은 구조체 기반의 자료형으로 다음과 같다.

```c
struct timeval {
  long tb_sec; // second 
  long tb_usec; // microseconds
}
```

이 변수의 주소값은 select 함수의 마지막 인자로 전달하게 되면 파일 디스크립터에 변화가 발생하지 않아도 지정된 시간이 지나면 함수가 반환한다. 단1 이렇게 해서 반환이 되는 경우, select 함수는 0을 반환한다. 그리고 타임아웃 설정하고 싶지 않을 경우 NULL 을 전달한다.

## select 함수호출 이후 결과확인
0이 아닌 양수가 반환되면 그 수만큼 파일 디스크립터에 변화가 발생했음을 의미하는데 그렇다면 select 함수가 양의 정수를 반환한 경우 변화가 발생한 파일 디스크립터는 어떻게 알아낼 수 있을까? select 함수의 두 번재 , 세 번째 그리고 네번째 인자로 전달된 fd_set형 변수에 다음 그림에서 보이는 변화가 발생하기 때문에 어렵지 않게 알아낼 수 있다.

![alt text](/image/18.png)

위 그림에서 보이듯이 select 함수호출이 완료되고 나면 select 함수의 인자로 전달된 fd_set형 변수에는 변화가 생긴다. 1로 설정된 모든 비트가 다 0으로 변경되지만, 변화가 발생한 파일 디스크립터에 해당하는 비트만 그대로 1로 남아있게 된다. 때문에 여전히 1로 남아있는 위치의 파일 디스크립터에서 변화가 발생했다고 판단이 가능하다.

## select 함수 프로그래밍
```c
#include <stdio.h>
#include <sys/select.h>
#include <sys/time.h>
#include <unistd.h>

#define BUF_SIZE 30

int main(int argc, char const *argv[])
{
  fd_set reads , temps;
  int result , str_len;
  char buf[BUF_SIZE];
  struct timeval timeout;

  FD_ZERO(&reads); // 모든 비트 초기화
  FD_SET(0 , &reads); // 0 is standard input(console) | 파일 디스크립터 0 은 console 창임

  /*
    timeout.tv_sec = 5;
    timeout.tv_usec = 5000;
  */

 while(1) {
  temps = reads;
  timeout.tv_sec = 5;
  timeout.tv_usec = 0;
  result = select(1 , &temps , 0 , 0, &timeout);

  if(result == -1) {
    puts("select() error!");
    break;
  }
  else if (result == 0) {
    puts("time out!");
  } else {
    if(FD_ISSET(0 , &temps)) {
      str_len = read(0 , buf , BUF_SIZE);
      buf[str_len] = 0;
      printf("message from console : %s" , buf);
    }
  }
 }

  return 0;
}
```

```
// 콘솔에 입력했을 시
[root@localhost chapter12]# ./select 
hi
message from console : hi
what the fuck
message from console : what the fuck

// 아무것도 안할때
[root@localhost chapter12]# ./select 
time out!
time out!
time out!
```

## 멀티플렉싱 에코 서버의 구현
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <sys/select.h>
#include <sys/time.h>

#define BUF_SIZE 100
void error_handling(char * message);

int main(int argc, char const *argv[])
{
  int serv_sock , clnt_sock;
  struct sockaddr_in serv_adr , clnt_adr;
  struct timeval timeout;
  fd_set reads , temps;

  socklen_t adr_sz;
  int fd_max , str_len , fd_num , i;
  char buf[BUF_SIZE];

  if(argc != 2) {
    printf("Usage : %s <port> \n" , argv[0]);
    exit(1);
  }

  serv_sock = socket(PF_INET , SOCK_STREAM , 0);
  memset(&serv_adr , 0 , sizeof(serv_adr));
  serv_adr.sin_family = AF_INET;
  serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
  serv_adr.sin_port = htons(atoi(argv[1]));

  if(bind(serv_sock , (struct sockaddr *) &serv_adr , sizeof(serv_adr)) == -1) 
    error_handling("bind() error!");
  if(listen(serv_sock , 5) == -1)
    error_handling("listen() error!");

  FD_ZERO(&reads); // 초기화
  FD_SET(serv_sock , &reads); 
  fd_max = serv_sock;

  while(1) {
    temps = reads;
    timeout.tv_sec = 5;
    timeout.tv_usec = 5000;

    if((fd_num = select(fd_max + 1, &temps , 0 , 0 , &timeout)) == -1) // except 시
      break;
    
    if(fd_num == 0) // timeout 시
      continue;

    for(i = 0; i < fd_max + 1; i++) {
      if(FD_ISSET(i , &temps)) {
        if(i == serv_sock) { // connection
          adr_sz = sizeof(clnt_adr);

          clnt_sock = accept(serv_sock , (struct sockaddr *) &clnt_adr , &adr_sz);

          FD_SET(clnt_sock , &reads);

          if(fd_max < clnt_sock)
            fd_max = clnt_sock;
            
          printf("connected client : %d \n" , clnt_sock);
        } 
        else { // read message
          str_len = read(i , buf , BUF_SIZE);
          if(str_len == 0) {// close request
            FD_CLR(i , &reads);
            close(i);
            printf("closed clientd : %d \n" , i);
          }
          else {
            write(i , buf , str_len); // echo!!
          }
        }
      }
    }
  }
  return 0;
}


void error_handling(char * message) {
  fputs(message , stderr);
  fputc('\n' , stderr);
  exit(1);
}
```

```
serv_sock fd : 3 
connected client : 4 
closed client : 4 
```

