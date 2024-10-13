## readv & writev 입출력 함수
이번에 소개할 readv & writev 입출력 함수는 데이터 송수신의 효율성을 향상시키는데 도움이 되는 함수들이다. 일단 사용법을 익혀보자

### readv & writev 함수의 사용
두 함수의 기능을 한마디로 정리하면 다음과 같다.

"데이터를 모아서 전송하고, 모아서 수신하는 기능의 함수"

즉, writev함수를 사용하면 여러 버퍼에 나뉘어 저장되어있는 데이터를 한번에 전송할 수 있고, 또 readv함수를 사용하면 데이터를 여러 버퍼에 나눠서 수신할 수 있다. 때문에 적절한 상황에서 사용을 하면 입출력 함수호출의 수를 줄일 수 있다.

먼저 writev함수이다.

```c
#include <sys/uio.h>

ssize_t writev (int __fd, const struct iovec *__iovec, int __count) // 성공시 바이트 수 실패시 -1

// filedes : 데이터 전송의 목적지를 나타내는 소켓의 파일 디스크립터 전달, 단 소켓에만 제한된 함수가 아니기 때문에 read 함수처럼 파일이나 콘손 대상의 파일 디스크립터도 전달 가능하다.
// iov : 구조체 iovec 배열의 주소값을 전달, 구조체 iovec의 변수에는 전송할 데이터의 위치 및 크기 정보가 담긴다.
// iovcnt : 두번째 인자로 전달된 주소 값이 가리키는 배열의 길이정보를 전달
```

그리고 iovec의 배열의 구조체는 다음과 같다.

```c
/* Structure for scatter/gather I/O.  */
struct iovec
  {
    void *iov_base;	/* Pointer to data.  */
    size_t iov_len;	/* Length of data.  */
  };
```

이렇듯 구조체 iovec은 전송할 데이터가 저장되어 있는 버퍼 (char형 배열)의 주소값과 실제 전송할 데이터의 크기 정보를 담기 위해 정의되었다.

![alt text](/image/22.png)

위 그림에서 writev의 첫 번째 인자 1은 파일 디스크립터를 의미하므로 콘솔에 출력이 이뤄지고, ptr은 전송할 데이터 정보를 모아둔 iovec 배열을 가리키는 포인터이다. 또한 세번째 인자가 2이기때문에 ptr이 가리키는 주소를 시작으로 총 두개의 iovec변수를 참조하여 그 두 변수가 가리키는 버퍼에 저장된 데이터의 전송이 진행된다. 그럼 이번에는 위 그림의 iovec 구조체 배열을 자세히 관찰하자 

ptr[0]의 (배열 첫 번째 요소의) iov_base 는 A로 시작하는 문자열을 가리키면서 iov_len 이 3이므로 ABC가 전송된다. 그리고 ptr[1]의 (배열 두 번째 요소의) iov_base는 숫자 1을 가리키며 iov_len 이 4이므로 1234가 이어서 전송된다.

```c
#include <stdio.h>
#include <sys/uio.h>

int main(int argc, char const *argv[])
{
  struct iovec vec[2];
  char buf1[] = "ABCDEFG";
  char buf2[] = "1234567";

  int str_len;

  vec[0].iov_base = buf1;
  vec[0].iov_len = 3;
  vec[1].iov_base = buf2;
  vec[1].iov_len = 4;

  str_len = writev(1 , vec , 2);
  puts("");
  printf("Write bytes : %d \n" , str_len);
  return 0;
}
```

```
[root@localhost chapter13]# ./writev 
ABC1234
Write bytes : 7 
```

## readv 함수
이어서 readv 함수이다. readv 함수는 writev 함수의 반대로 생각하면 된다.

```c
#include <sys/uio.h>

ssize_t readv (int __fd, const struct iovec *__iovec, int __count) // 성공시 바이트 수 실패시 -1

// filedes : 데이터 전송의 목적지를 나타내는 소켓의 파일 디스크립터 전달, 단 소켓에만 제한된 함수가 아니기 때문에 read 함수처럼 파일이나 콘손 대상의 파일 디스크립터도 전달 가능하다.
// iov : 구조체 iovec 배열의 주소값을 전달, 구조체 iovec의 변수에는 전송할 데이터의 위치 및 크기 정보가 담긴다.
// iovcnt : 두 번째 인자로 전달된 주소 값이 가리키는 배열의 길이정보 전달
```

바로 예제를 보자

```c
#include <sys/uio.h>
#include <stdio.h>

#define BUF_SIZE 100

int main(int argc, char const *argv[])
{
  struct iovec vec[2];

  char buf1[BUF_SIZE] = {0 ,};
  char buf2[BUF_SIZE] = {0 ,};
  int str_len;

  vec[0].iov_base = buf1;
  vec[0].iov_len = 5;
  vec[1].iov_base = buf2;
  vec[1].iov_len = BUF_SIZE;

  str_len = readv(0 , vec , 2); // 0 : console
  printf("Read bytes : %d \n" , str_len);
  printf("First message : %s \n" , buf1);
  printf("Second message : %s \n" , buf2);
  return 0;
}
```

```
[root@localhost chapter13]# ./readv
I like TCP/IP socket programming~
Read bytes : 34 
First message : I lik 
Second message : e TCP/IP socket programming~
```

실행 결과를 보면 7행에 선언된 배열 vec 의 정보를 참조해서 데이터가 저장되었음을 알 수 있다.

## reacv & writev 함수의 적절한 사용
두 함수를 사용하기에 적절한 상황은 사용할 수 있는 모든 경우가 적절한 상황이다. 예를 들어 전송해야할 데이터가 여러개의 버퍼 (배열)에 나뉘어 있는 경우, 모든 데이터의 전송을 위해서는 여러 번의 write 함수호출이 요구되는데 이를 딱 한번의 wirtev 함수 호출로 대신할 수 있으니 당연히 효율적이다. 마찬가지로 입력버퍼에 수신된 데이터를 여러 저장소에 나눠서 읽어 들이고 싶은 경우에도 여러 번 read 함수를 호출하는 것보다 딱 한번 readv 함수를 호출하는 것이 보다 효율적이다.

그러나 전송되는 패킷의 수를 줄일 수 있다는데 더 큰 의미가 있다. 우리가 구현한 서버에서 성능향상을 위해 Nagle 알고리즘을 명시적으로 중지시킨 상황을 예로 들어보자. 사실 writev함수는 Nagle 알고리즘이 중지된 상황에서 더 활용의 가치가 높다.

![alt text](/image/23.png)

위 그림은 전송해야할 데이터가 세 곳의 영역으로 나뉘어 저장된 상황에서의 데이터 전송을 예로 들고 있다. 이 상황에서 write 함수를 사용할 경우 총 세번이 필요하다. 그런데 속도향상을 목적으로 이미 Nagle 알고리즘이 중지된 상황이라면 총 세 개의 패킷이 생성되어 전송될 확률이 높다. 반면 writev함수를 사용할 경우 한번에 모든 데이터를 출력버퍼로 밀어넣기 때문에 하나의 패킷만 생성되어 전송될 확률이 높다. 때문에 writev 와 readv 함수의 호출이 유용한 것이다.

그렇다면 한가지 더 생각해보자 여러 영역에 나뉘어 있는 데이터를 전송순서에 맞춰 하나의 큰 배열에 옮겨다 놓고 한번의 write 함수호출을 통해서 전송을 하면 writev 함수를 호출한 것과 같은 결과를 얻을 수 있다.
그러나 그것보다는 writev 함수를 사용하는 것이 여러모로 편리하다.

