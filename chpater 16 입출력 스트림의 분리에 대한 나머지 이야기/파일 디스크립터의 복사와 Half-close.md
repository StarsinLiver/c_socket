## 파일 디스크립터의 복사와 Half-close

이전 '입력 스트림과 출력 스트림의 분리' 를 참조

## 스트림 종료 시 Half-close 가 진행되지 않은 이유

다음 그림은 예제 sep_serv.c 에서 보인 두 개의 FILE 포인터와 파일 디스크립터, 그리고 소켓의 관계를 보이고 있다.

![alt text](/image/26.png)

위 그림에서 보이듯이 소켓이 소멸되면 더 이상 데이터의 송수신은 불가능한 상태가 된다. 그렇다면 출력은 불가능 하지만 입력은 가능한 Half-close 상태는 어떻게 만들어야 할까? 이는 간단하게 FILE 포인터를 생성하기에 앞서 파일 디스크립터를 복사하면 된다.

복사를 통해서 파일 디스크립터를 하나 더 만든 다음 각각의 파일 디스크립터를 통해서 읽기모드 FILE 포인터와 스기모드 FILE 포인터를 만들면 Half-close 의 환경은 마련된다. 이는 소켓과 파일 디스크립터의 사이에는 다음 관계가 있기 때문이다.

"모든 파일 디스크립터가 소멸되어야 소켓도 소멸된다."

![alt text](/image/27.png)

위 그림에서 보이듯이 fclose 함수호출 후에도 아직 파일 디스크립터가 하나 더 남아있기 때문에 소켓은 소멸되지 않는다. 그렇다면 이 상태를 Half-clsoe 의 상태로 볼 수 있을까? 그건 아니다. Half-close 는 별도의 과정을 거쳐야 한다.

위 그림을 보면 Half-close 가 진행된 것 처럼 보이지만 자세히 보면 파일 디스크립터가 하나 남아있다. 그리고 파일 디스크립터는 입력 및 출력이 모두 가능하다. 때문에 상대 호스트로 EOF가 전송되지 않음은 물론이거니와 파일 디스크립터를 이용하면 출력도 여전히 가능한 상태이다. 따라서 잠시 후의 모델을 바탕으로 EOF 가 전송되는 Half-close 를 진행하는 방법을 별도로 설명할 것이다. 그러니 이에 대한 내용은 잠시 뒤로하고 먼저 파일 디스크립터의 복사 방법부터 고민해보자.

## 파일 디스크립터의 복사

위에서 말한 파일 디스크립터의 복사는 fork 함수호출 시에 진행되는 복사와는 차이가 있다. fork 함수 호출시 진행되는 복사는 프로세스를 통째로 복사하는 상황엣 이뤄지기 때문에 하나의 프로세스에 원본과 복사본이 모두 존재하지 않는다. 그러나 여기서 말하는 복사는 프로세스의 생성을 동반하지 않는, 원본과 복사본이 하나의 프로세스 내에 존재하는 형태의 복사를 뜻한다.

(그림 359) 의 그림은 하나의 프로세스 내에 동일한 파일에 접근할 수 있는 파일 디스크립터가 두 개 존재하는 상황을 설명한다. 물론 파일 디스크립터는 값이 중복될 수 없으므로 두 파일 디스크립터의 정수 값은 5와 7로 서로 다르다. 이러한 형태로 파일 디스크립터를 구성하려면 파일 디스크립터를 복사해야한다. 즉, 여기서 말하는 복사는 다음과 같이 정의할 수 있다.

"동일한 파일 또는 소켓의 접근을 위한 또 다른 파일 디스크립터의 생성"

흔히 복사라고 하면 있는 그대로, 파일 디스크립터의 정수 값까지 복사한다고 생각하기 쉬운데, 여기서 말하는 복사는 그러한 의미의 복사가 아니다.

## dup & dup2

그럼 이번에는 파일 디스크립터의 복사방법에 대해서 이야기 해보자. 파일 디스크립터의 복사는 다음 두 함수 중에 하나를 이용해서 진행한다.

```c
#include <unistd.h>

int dup(int fildes); // 성공시 복사된 파일 디스크립터 , 실패 시 -1 반환
int dup2(int fildes , int fildes2); // 성공시 복사된 파일 디스크립터 , 실패 시 -1 반환

// fildes : 복사할 파일 디스크립터 전달
// fildes2 : 명시적으로 지정할 파일 디스크립터의 정수 값 전달
```

dup2 함수는 복사된 파일 디스크립터의 정수 값을 명시적으로 지정할 때 사용한다. 이 함수의 인자로 0보다 크고 프로세스 당 생성할 수 있는 파일 디스크립터의 수보다 작은 값을 전달하면 해당 값을 복사되는 파일 디스크립터의 정수 값으로 지정해준다.

다음 예제에서는 시스템에 의해서 자동으로 열리는 표준출력을 의미하는 파일 디스크립터 1을 복사하여 복사된 파일 디스크립터를 이용해서 출력을 진행한다. 참고로 자동으로 열리는 파일 디스크립터 0,1,2 역시 소켓 기반의 파일 디스크립터와 차이가 없으므로 dup 함수의 기능확인을 목적으로 사용하기에 충분하다.

```c
#include <stdio.h>
#include <unistd.h>

int main(int argc, char const *argv[])
{
  int cfd1, cfd2;
  char str1[] = "Hi~ \n";
  char str2[] = "It's nice day~ \n";

  cfd1 = dup(1);
  cfd2 = dup2(cfd1, 7);

  printf("fd1 = %d , fd2 = %d \n", cfd1, cfd2);
  write(cfd1, str1, sizeof(str1));
  write(cfd2, str2, sizeof(str2));

  close(cfd1);
  close(cfd2);

  write(1, str1, sizeof(str1));
  close(1);
  write(1, str2, sizeof(str2));

  return 0;
}
```

```
fd1 = 3 , fd2 = 7
Hi~
It's nice day~
Hi~
```

## 파일 디스크립터의 복사 후 스트림의 분리

이제 끝으로 예제 sep_serv.c 와 sep_clnt.c 가 정상동작하도록 변경할 차례이다. 그런데 여기서 말하는 정상동작은 서버 측의 Half-close 진행으로 클라이언트가 전송하는 마지막 문자열이 수신되는 것을 의미한다. 물론 이를 위해서는 서버 측에서의 EOF 전송을 동반해야 한다. 그런데 EOF 의 전송은 어렵지 않기 때문에 예제를 통해서 함께 보자

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#define BUF_SIZE 1024

int main(int argc, char const *argv[])
{
  int serv_sock, clnt_sock;
  FILE *readfp;
  FILE *writefp;

  struct sockaddr_in serv_adr, clnt_adr;
  socklen_t clnt_adr_sz;
  char buf[BUF_SIZE] = {0};

  serv_sock = socket(PF_INET, SOCK_STREAM, 0);
  memset(&serv_adr, 0, sizeof(serv_adr));
  serv_adr.sin_family = AF_INET;
  serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
  serv_adr.sin_port = 9190;

  bind(serv_sock, (struct sockaddr *)&serv_adr, sizeof(serv_adr));
  listen(serv_sock, 5);
  clnt_adr_sz = sizeof(clnt_adr);

  clnt_sock = accept(serv_sock, (struct sockaddr *)&clnt_adr, &clnt_adr_sz);

  readfp = fdopen(clnt_sock, "r");
  writefp = fdopen(dup(clnt_sock), "w");

  fputs("FROM SERVER : HI ~ Client? \n", writefp);
  fputs("I love all of the world \n", writefp);
  fputs("You are awesome! \n", writefp);
  fflush(writefp);

  shutdown(fileno(writefp), SHUT_WR);
  fclose(writefp);

  fgets(buf, sizeof(buf), readfp);
  fputs(buf, stdout);
  fclose(readfp);
  return 0;
}
```

```
[root@localhost chapter16]# ./sep_serv2
FROM CLIENT : Thank you!
```

실행결관느 Half-close 상태에서 클라이언트로 EOF 가 전송되었음을 증명하고 있다. 따라서 이 예제를 통해서 다음 사실을 정리하기 바란다.

"복사된 파일 디스크립터의 수에 상관없이 EOF 의 전송을 동반하는 Half-close 를 진행하기 위해서는 shutdown 함수를 호출해야 한다."

이러한 shutdown 함수의 기능은 Chapter 10의 예제 echo_mpclient.c 에서도 활용한 바 있다. 당시에도 fork 함수 호출을 통해서 두 개의 파일 디스크립터가 존재하는 상황에서의 EOF 전송이 필요했었고 이를 위해서 shutdown 함수를 호출했었다.
