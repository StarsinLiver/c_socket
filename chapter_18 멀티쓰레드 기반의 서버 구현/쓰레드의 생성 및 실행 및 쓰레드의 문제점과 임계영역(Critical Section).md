## 쓰레드의 생성 및 실행

POSIX 란 Portable Operating System Interface for Computer Environment 의 약자로써 UNIX 계열 운영체제간에 이식성을 높이기 위한 표준 API 규격을 뜻한다. 그리고 이어서 설명하는 쓰레드의 생성방법은 POSIX 에 정의된 표준을 근거로 한다. 리눅스 뿐만 아니라 유닉스 계열의 운영체제에서도 대부분 적용이 가능하다.

## 쓰레드의 생성과 실행흐름의 구성

쓰레드는 별도의 실행 흐름을 갖기 때문에 쓰레드만의 main 함수를 별도로 정의해야한다. 그리고 이 함수를 시작으로 별도의 실행흐름을 형성해 줄 것을 운영체제에게 요청해야 하는데, 이를 목적으로 호출하는 함수는 다음과 같다.

```c
#include <pthread.h>

int pthread_create (pthread_t *__restrict __newthread, const pthread_attr_t *__restrict __attr, void *(*__start_routine) (void *), void *__restrict __arg) __THROWNL __nonnull ((1, 3)); // 성공 시 0 , 실패 시 0 이외의 값 반환

// thread : 생성할 쓰레드의 ID 저장을 위한 변수의 주소 값 전달. 참고로 쓰레드는 프로세스와 마찬가지로 쓰레드의 구분을 위한 ID가 부여됨
// attr : 쓰레드에 부여할 특성정보의 전달을 위한 매개변수 , NULL 전달 시 기본적인 특성의 쓰레드가 생성됨
// start_routine : 쓰레드의 main 함수 역할을 하는 별도 실행흐름의 시작이 되는 함수의 주소 값(함수 포인터) 전달
// arg : 세번째 인자를 통해 등록된 함수가 호출될 때 전달할 인자의 정보를 담고 있는 변수의 주소값 전달
```

그런 간단한 예를 통해 알아보자

```c
//
// Created by root on 24. 11. 10.
//

#include <unistd.h>
#include <stdio.h>
#include <pthread.h>

void* thread_main(void * arg);

int main() {
  pthread_t t_id;
  int thread_param = 5;

  if(pthread_create(&t_id , NULL , thread_main , (void*)  &thread_param) != 0) {
    puts("pthread_create() error");
    return -1;
  }

  sleep(10); puts("end of main");

  return 0;
}

void* thread_main(void* arg) {
  int i ;
  int cnt = *((int*)arg);
  for(i = 0; i < cnt; i++) {
      sleep(1); puts("running thread");
  }
  return NULL;
}
```

```
[root@localhost chapter19]# gcc thread1.c -o tr1 -lpthread
[root@localhost chapter19]# ./tr1
running thread
running thread
running thread
running thread
running thread
end of main
```

쓰레드 관련코드는 컴파일 시 -lpthread 옵션을 추가해서 쓰레드 라이브러리의 링크를 별도로 지시해야한다. 그래야 헤더파일 pthread.h 에 선언된 함수들을 호출할 수 있다.

메인 함수가 종료되면 쓰레드도 같이 종료된다.

sleep함수의 호출을 통해서 쓰레드의 실행을 관리하는 것은 사실상 불가능하다. 따라서 join 함수가 필수다.

```c
#include <pthread.h>

// int pthread_join (pthread_t __th, void **__thread_return); <-- gcc 8 버전
int pthread_join(pthread_t thread , void** status); // 성공 시 0 , 실패 시 0 이외의 값 반환

// thread : 이 매개변수에 전달되는 ID의 쓰레드가 종료될 때까지 함수는 반환하지 않는다.
// status : 쓰레드의 main  함수가 반환하는 값이 저장될 포인터 변수의 주소 값을 전달한다.
```

간단히 말해서 위 함수는 첫번째 인자로 전달되는 ID 의 쓰레드가 종료될때까지 이 함수를 호출한 프로세스(또는 쓰레드)를 대기상태에 둔다. 뿐만아니라, 쓰레드의 main 함수가 반환하는 값까지 얻을 수 있으니 그만큼 유용한 함수이다.

```c
//
// Created by root on 24. 11. 10.
//
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>

void* thread_main(void* arg);

int main (void) {
    pthread_t t_id;
    int thread_param = 5;

    void* thr_ret;

    if(pthread_create(&t_id , NULL , thread_main, (void*) &thread_param) != 0) {
        puts("pthread_create() error!");
        return -1;
    }

    if(pthread_join(t_id , &thr_ret) != 0) {
        puts("pthread_join() error!");
        return -1;
    }

    printf("Thread return message : %s \n" , (char*) thr_ret);
    free(thr_ret);
    return 0;
}

void* thread_main(void* arg) {
    int i;
    int cnt = *((int *) arg);
    char * msg = (char*) malloc(sizeof(char) * 50);
    strcpy(msg , "Hello, I'am thread~ \n");

    for(i = 0; i < cnt; i++) {
        sleep(1); puts("runiing thread");
    }
    return msg;
}
```

```
[root@localhost chapter19]# gcc thread2.c -o tr2 -lpthread
[root@localhost chapter19]# ./tr2
runiing thread
runiing thread
runiing thread
runiing thread
runiing thread
Thread return message : Hello, I'am thread~
```

## 임계영역 내에서 호출이 가능한 함수

앞서 보인 예제에서는 쓰레드를 하나만 생성했었다. 그러나 이제부터 동시에 둘 이상의 쓰레드를 생성해 볼 것이다. 물론 쓰레드를 하나 생성하건 둘을 생성하건 그 방법에 있어서 차이는 없다. 그러나 쓰레드의 실행과 관련해서 주의해야 할 사실이 하나있다. 함수 중에는 둘 이상의 쓰레드가 동시에 호출하면 문제를 일으키는 함수가 있다. 이는 함수 내에 '임계영역 (Critical Section)' 이라 불리는, 둘 이상의 쓰레드가 동시에 실행하면 문제를 일으키는 문장이 하나 이상 존재하는 함수이다.

일단은 둘 이상의 쓰레드가 동시에 실행하면 문제를 일으키는 코드블록을 가리켜 임계영역이라 한다는 사실만 기억하자 이러한 임계영역의 문제와 관련해서 ㅎ마수는 다음 두 가지 종류로 구분이 된다.

```
1. 쓰레드에 안전한 함수 (Thread-safe function)
2. 쓰레드에 불안전한 함수 (Thread-unsafe function)
```

여기서 쓰레드에 안전한 함수는 둘 이상의 쓰레드에 의해서 동시에 호출 및 실행되어도 문제가 되지 않는 함수다. 반대로 쓰레드에 불안전한 함수는 동시호출시 문제가 발생할 함수를 뜻한다. 다만, 둘 다 영역을 둘 이상의 쓰레드가 동시에 접근해도 문제를 일으키지 않도록 적절한 조치가 이뤄져야 한다.

다행히 기볹거으로 제공되는 대부분의 표준함수들은 쓰레드에 안전하다. 그러나 그보다 더 다행인것은 쓰레드에 안전한 함수와 불안전한 함수의 구분을 우리가 직접할 필요가 없다는것에 있다. 왜냐하면 쓰레드에 불안전한 함수가 정의되어 있는 경우 같은 기능을 갖는 쓰레드에 안전한 함수가 정의되어 있기 때문이다.

chapter 8 에서 소개한 다음 함수는 쓰레드에 안전하지 못하다.

```c
struct hostent * gethostbyname (const char * hostname);
```

때문에 동일한 기능을 제공하면서 쓰레드에 안전한 다음 함수가 이미 정의되어 있다.

```c
struct hostent * gethostbyname_r (const char *name , struct hostent *result , char * buffer , intbuflen , int * h_errnop);
```

일반적으로 쓰레드에 안전한 형태로 재 구현된 함수의 이름에는 \_r 이 붙는다. 그렇다면 둘 이상의 쓰레드가 동시에 접근 가능한 코드 블록에서는 gethostbyname 함수를 대신해서 gethostbyname_r 을 호출해야 할까? 물론이다! 하지만 이는 프로그래머에게 엄청난 수고를 요구하는 것이다. 그런데 다행히도 다음의 방법으로 이를 자동화 할 수 있다. 즉, 다음과 같은 방법을 통해서 gethostbyname 함수의 호출문을 gethostbyname_r 함수의 호출문으로 변경할 수 있다.

"헤더파일 선언 이전에 매크로 \_REENTRANT 를 정의한다."

gethostbyname 함수와 gethostbyname_r 함수가 이름에서뿐만아니라, 매개변수의 선언에서도 차이가 난다는 사실을 알았으니 이것이 얼마나 매력적인지 알 수 있을 것이다. 그리고 위의 매크로 정의를 위해서 굳이 소스코드에 #define 문장을 추가할 필요는 없다. 다음과 같이 컴파일 시 -D REENTRANT 의 옵션을 추가하는 방식으로도 매크로를 정의할 수 있다.

```
# gcc -D_REEENTRANT mythread.c -o mthread -lpthread
```

## 워커(Worker) 쓰레드 모델

지금까지는 쓰레드의 개념 및 생성방법의 이해를 목적으로 간단히 작성했다. 이제 둘 이상의 쓰레드가 생성되는 예제를 작성해보자

1부터 10까지의 덧셈결과를 출력하는 예제를 만들어 보자 그런데 main 함수에서 덧셈을 진행하는 것이 아니라, 두 개의 쓰레드를 생성해서 하나는 1부터 5가지 다른 하나는 6부터 10까지 덧셈하도록 하고 main 함수에서는 단지 연산결과를 출력하는 형태로 작성해 보고자 한다.

위의 유형의 프로그래밍 모델을 가리켜 '워커 쓰레드 (Worker thread) 모델' 이라고 한다.

```c
//
// Created by root on 24. 11. 10.
//

#include <stdlib.h>
#include <stdio.h>
#include <pthread.h>

void* thread_summation(void * arg);
int sum = 0;
int main(void) {
    pthread_t id_t1 , id_t2;
    int range1[] = {1 , 5};
    int range2[] = {6 , 10};

    pthread_create(&id_t1, NULL , thread_summation , (void*) range1);
    pthread_create(&id_t2, NULL , thread_summation , (void*) range2);

    pthread_join(id_t1 , NULL);
    pthread_join(id_t2 , NULL);

    printf("result %d \n" , sum);
    return 0;
}

void* thread_summation(void * arg) {
     int start = ((int*) arg)[0];
     int end = ((int*) arg)[1];

     while(start <= end) {
         sum += start;
         start++;
     }
     return NULL;
}
```

```
[root@localhost chapter19]# gcc -D_REENTRANT thread3.c -o tr3 -lpthread
[root@localhost chapter19]# ./tr3
result 55
```

예제를 하나 더 제시하겠다. 이는 위의 예제와 거의 비슷하다 다만 앞서 설명한 임계영역과 관련해서 오류의 발생소지를 더 높였을 뿐이다. 이 정도 예제면 아무리 시스템의 성능이 좋아도 어렵지 않게 오류의 발생을 확인할 수 있을 것이다.

```c
/* 오류 발생 코드 */
//
// Created by root on 24. 11. 10.
//
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <pthread.h>

#define NUM_THREAD 100

void * thread_inc (void* arg);
void * thread_des (void* arg);
long long num = 0; // 64 bit 정수 자료형

int main(void) {
    pthread_t thread_id[NUM_THREAD];
    int i;

    printf("sizeof long long : %d \n" , sizeof(long long));

    for(i = 0; i < NUM_THREAD; i++) {
        if(i%2)
            pthread_create(&(thread_id[i]) , NULL , thread_inc , NULL);
        else
            pthread_create(&(thread_id[i]) , NULL , thread_des , NULL);
    }

    for(i = 0; i < NUM_THREAD; i++) {
        pthread_join(thread_id[i] , NULL);
    }

    printf("result : %lld \n" , num);
    return 0;
}

void * thread_inc (void* arg) {
    int i;
    for(i = 0; i < 50000000; i++) {
        num += 1;
    }
    return NULL;
}
void * thread_des (void* arg) {
    int i;
    for(i = 0; i < 50000000; i++) {
        num -= 1;
    }
    return NULL;
}

```

```
[root@localhost chapter19]# gcc thread4.c -D_REENTRANT -o th4 -lpthread
[root@localhost chapter19]# ./th4
sizeof long long : 8
result : 11313750
[root@localhost chapter19]# ./th4
sizeof long long : 8
result : -8221396
```

실행할 때마다 매번 그 결과도 다르다. 어찌되었든 이는 쓰레드를 활용하는데 있어서 큰 문제임이 틀림없다.

## 쓰레드의 문제점과 임계영역(Critical Section)

앞서 '쓰레드의 생성 및 실행' 에서의 thread4.c 의 문제점을 해결해보자

## 하나의 변수에 둘 이상의 쓰레드가 동시에 접근하는 것이 문제

thread4.c 의 문제점은 다음과 같다.

"전역변수 num 에 둘 이상의 쓰레드가 함께 (동시에) 접근하고 있다."

여기서 말하는 접근이란 주로 값의 변경을 뜻한다. 그런데 보다 다양한 상황에서 문제가 발생할 수 있기 때문에 문제의 원인이 무엇인지 정확히 이해해야 한다. 그리고 예제에서는 접근대상이 전역변수였지만 이는 전역변수였기 때문에 발생한 문제가 아니다. 어떠한 메모리 공간이라도 동시에 접근을 하면 문제가 발생할 수 있다.

"쓰레드들은 CPU의 할당시간을 나눠서 실행하게 된다면서요. 그러면 실제로는 동시접근이 이뤄지지 않을 것같은데요."

물론 여기서 말하는 동시접근은 우리가 생각하는 동시접근과는 약간 차이가 있다. 그래서 하나의 예를 통해서 동시접근이 무엇인지, 그리고 이것이 왜 문제가 되는지 알아보자.
먼저 변수에 저장된 값을 1씩 증가시키는 연산을 두 개의 쓰레드가 진행하려는 상황이라고 가정해보자

위 상황에서 thread1 이 변수 num 에 저장된 값을 100으로 증가시켜놓은 다음에 이어서 thread2 가 변수 num 에 접근하면 우리의 예상대로 변수 num 에는 101이 저장된다.
다음그림은 thread1 이 변수 num 에 저장된 값을 완전히 증가시킨 상황을 보인다.

![alt text](/image/31.png)

그런데 위 그림에서 한가지 주목해야할 사실이 있다. 그것은 값의 증가방식이다. 값의 증가는 CPU를 통한 연산이 필요한 작업이다. 따라서 그냥 변수 num 에 저장된 값이, 변수 num 에 저장된 상태로 증가하지 않는다.
이 변수에 저장된 값은 thread1 에 의해서 우선 참조가 된다. 그리고 thread1 은 이 값을 CPU에 전달해서 1이 증가된 100을 얻는다. 마지막으로 연산이 완료된 값을 변수 num 에 다시 저한다. 이렇게 num 에 100이 저장된다. 그럼 이어서 thread2 도 값을 증가시켜보자

![alt text](/image/32.png)

이렇게 해서 변수 num 에는 101이 저장된다. 그런데 이는 매우 이상적인 상황을 묘사한 것이다.

thread1 이 변수 num 에 저장된 값을 완전히 증가시키기 전에라도 얼마든지 thread2 로 CPU의 실행이 넘어갈 수 있기 때문이다.

#### 밑과 같이 문제가 생길 수 있다.

![alt text](/image/33.png)
![alt text](/image/34.png)
![alt text](/image/35.png)

바로 이것이 '동기화 (Synchronization)' 이다.

## 임계영역은 어디?

"함수 내에 둘 이상의 쓰레드가 동시에 실행하면 문제를 일으키는 하나 이상의 문장으로 묶여있는 코드블록"

전역변수 num을 임계영역으로 보아야 할까? 아니다! 이는 문제를 일으키는 문장이 아니지 않은가 따라서 thread4 에서는 밑과 같다.

```c
void * thread_inc (void* arg) {
    int i;
    for(i = 0; i < 50000000; i++) {
        num += 1; /* <------- 임계영역 */
    }
    return NULL;
}
void * thread_des (void* arg) {
    int i;
    for(i = 0; i < 50000000; i++) {
        num -= 1; /* <------- 임계영역 */
    }
    return NULL;
}
```

문제가 발생하는 상황은 다음과 같이 세가지 형태로 나눠서 정리할 수 있다.

```
1. 두 쓰레드가 동시에 thread_inc 함수를 실행하는 경우
2. 두 쓰레드가 동시에 thread_des 함수를 실행하는 경우
3. 두 쓰레드가 각각 thread_inc 함수와 thread_des 함수를 동시에 실행하는 경우
```

"쓰레드 1이 thread_inc 함수의 문장 num += 1 을 실행할 때, 동시에 쓰레드 2가 thread_des 함수의 문장 num -= 1 을 실행하는 상황"

이렇듯 임계영역은 서로 다른 두 문장이 각각 다른 스레드에 의해서 동시에 실행되는 상황에서도 만들어 질 수 있다. 바로 그 두 문장이 동일한 메모리 공간에 접근을 한다면 말이다.
