## 쓰레드 동기화

이전의 문제를 해결하기 위한 해결책을 '쓰레드 동기화 (Synchronization)' 이라 한다. 이것이 무엇인지 살펴보자

## 동기화의 두 가지 측면

쓰레드의 동기화는 쓰레드의 접근순서 때문에 발생하는 문제점의 해결책을 뜻한다. 그런데 동기화가 필요한 상황은 다음 두 가지 측면에서 생각해 볼 수 있다.

```
1. 동일한 메모리 영역으로의 동시접근이 발생하는 상황
2. 동일한 메모리 영역에 접근하는 쓰레드의 실행순서를 지정해야 하는 상황
```

첫 번째 언급한 상황은 이미 충분히 설명되었으니, 두번째 언급한 상황에 대해서 이야기 해 보겠다. 이는 쓰레드의 '실행순서 컨트롤(Control)'에 관련된 내요잉다. 예를 들어서 스레드 A와 B가 있다고 가정해보자. 그런데 스레드 A는 메모리 공간에 값을 가져다 놓는 (저장하는) 역할을 담당하고, 쓰레드 B는 이 값을 가져가는 역할을 담당한다고 가정해보자. 이러한 경우에는 쓰레드 A가 약속된 메모리 공간에 먼저 접근해서 값을 저장해야 한다. 혹시라도 쓰레드 B가 먼저 접근해서 값을 가져가면 잘못된 결과로 이어질 수 있다.
이렇듯 실행수서의 컨트롤이 필요한 상황에서도 이어서 설명하는 동기화 기법이 활용된다.

우리는 '뮤텍스 (Mutex) 와 세마포어 (Semaphore)' 라는 두 가지 동기화 기법에 대해서 공부할 것이다. 그런데 이 둘은 개념적으로 매우 유사하다. 따럿 뮤텍스를 이해하고 나면 세마포어는 쉽게 이해할 수 있다. 뿐만 아니라 대부부느이 동기화 기법이 유사하기 때문에 여기서 설명하는 내용을 잘 이해하면 윈도우 기반의 동기화 기법도 쉽게 이해 및 활용이 가능하다.

## 뮤텍스 (Mutex)

뮤텍스란 'Mutual Exclusion'의 줄임말로써 쓰레드의 동시접근을 허용하지 않는다는 의미가 있다.

```
동수 : 똑똑! 안에 누구계세요?
응수 : 네 지금 열심히 볼일 보고 있습니다.

동수 : 똑똑!
응수 : 네 곧 나갑니다.
```

현실세계에서의 임계영역은 화장실이다. 화장실에 둘 이상의 사람이 동시에 들어갈 순 없지 않은가? 여기서 일어나는 모든 일은 임계영역의 동기화에서 거의 그대로 표현된다. 화장실 사용의 일반적인 규칙을 보자

```
1. 화장실의 접근보호를 위해서 들어갈 때 문을 잠그고 나올때 문을 연다
2. 화장실이 사용 중이라면, 밖에서 대기해야 한다.
3. 대기중인 사람이 둘 이상 될 수 있고, 이들은 대기순서에 따라서 화장실에 입성(?) 한다.
```

마찬가지로 쓰레드도 임계영역의 보호를 위해서는 위의 규칙이 반영되어야 한다.

위의 시스템을 자물쇠 시스템이라고 한다. 즉, 화장실에 들어갈 때 문을 잠그고, 나갈 때는 여는 그러한 자물쇠 시스템이 쓰레드의 동기화에 필요하다. 그리고 지금 설명하려는 뮤텍스는 매우 훌륭한 자물쇠 시스템이다.

그럼 이어서 뮤텍스라 불리는 자물쇠 시스템의 생성 및 소멸함수를 소개하겠다.

```c
#include <pthread.h>

int pthread_mutex_init (pthread_mutex_t *__mutex, const pthread_mutexattr_t *__mutexattr);
int pthread_mutex_destroy (pthread_mutex_t *__mutex)

// 성공 시 0 , 실패시 0 이외의 값 반환

// mutex : 뮤텍스 생성시에는 뮤텍의 참조 값 저장을 위한 변수의 주소 값 전달. 그리고 뮤텍스 소멸 시에는 소멸하고자 하는 뮤텍의 참조 값을 저장하고 있는 변수의 주소 값 전달.
// attr : 생성하는 뮤텍스의 특성정보를 담고 있는 변수의 주소 값 전달, 별도의 특성을 지정하지 않을 경우에는 NULL 전달
```

위 함수들을 통해서도 확일 할 수 있듯이, 자물쇠 시스템에 해당하는 뮤텍스의 생성을 위해서는 다음과 같이 pthread_mutex_t 형 변수가 하나 선언되어야 한다.

```c
pthread_mutex_t mutex;
```

그리고 이 변수의 주소 값은 pthread_mutex_init 함수 호출시 인자로 전달되어서 운영체제가 생성한 뮤텍스(자물쇠 시스템)의 참조에 사용된다. 때문에 pthread_mutex_destroy 함수호출 시에도 인자로 사용되는 것이다. 참고로 뮤텍스 생성시 별도의 특성을 지정하지 않아서 두 번째 인자로 NULL 을 전달하는 경우에는 매크로 PTHREAD_MUTEX_INITIALIZER 을 이용해서 다음과 같이 초기화 하는 것도 가능하다.

```c
pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
```

하지만 가급적이면 ptherad_mutex_init 함수를 이용한 초기화를 추천한다. 왜냐하면 매크로를 이용한 초기화는 오류발생에 대한 확인이 어렵기 때문이다. 그럼 이어서 뮤텍스를 이용해서 임계영역에 설치된 자물쇠를 걸어 잠그거나 풀때 사용되는 함수를 소개하겠다.

```c
#include <pthread.h>

int pthread_mutex_lock(pthread_mutex_t *mutex);
int pthread_mutex_unlock(pthread_mutex_t *mutex);

// 성공 시 0, 실패 시 0 이외의 값 반환
```

함수의 이름에서도 lock , unlock 이 있으니 쉽게 의미하는 바를 이해할 수 있다.

뮤텍스가 이미 생성된 상태에서는 다음의 형태로 임계영역을 보호하게 된다.

```c
pthread_mutex_lock(&mutex);
// 임계영역 시작
// ...
// 임계영역 끝
pthread_mutex_unlock(&mutex);
```

쉽게 말해서 lock, 그리고 unlock 함수를 이용해서 임계영역의 시작과 끝을 감싸는 것이다. 그러면 이것이 임계영역에 대한 자물쇠 역할을 하면서, 둘 이상의 쓰레드 접근을 허용하지 않게 된다. 한가지 더 기억해야 할 것은 임계영역을 빠져나가는 쓰레드가 pthread_mutex_unlock 함수를 호출하지 않는다면 임계영역으로의 진입을 위해 pthread_mutex_lock ㅎ마수는 블로킹 상태에서 빠져나가지 못하게 된다는 사실이다. 이를 두고 '데드락 (Dead-lock)' 이라고 하는데, 이러한 상황이 발생하지 않도록 주의해야 한다.

thread4.c 에서 보인 문제점을 해결해 보자

```c
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
pthread_mutex_t mutex;

int main(void) {
    pthread_t thread_id[NUM_THREAD];
    int i;

    pthread_mutex_init(&mutex , NULL);

    for(i = 0; i < NUM_THREAD; i++) {
        if(i%2)
            pthread_create(&(thread_id[i]) , NULL , thread_inc , NULL);
        else
            pthread_create(&(thread_id[i]) , NULL , thread_des , NULL);
    }

    for(i = 0; i < NUM_THREAD; i++) {
        pthread_join(thread_id[i] , NULL);
    }

    pthread_mutex_destroy(&mutex);
    printf("result : %lld \n" , num);
    return 0;
}

void * thread_inc (void* arg) {
    int i;
    pthread_mutex_lock(&mutex);
    for(i = 0; i < 50000000; i++) {
        num += 1;
    }
    pthread_mutex_unlock(&mutex);
    return NULL;
}
void * thread_des (void* arg) {
    int i;
    for(i = 0; i < 50000000; i++) {
      pthread_mutex_lock(&mutex);
        num -= 1;
      pthread_mutex_unlock(&mutex);
    }
    return NULL;
}
```

```
[root@localhost chapter19]# gcc -D_REENTRANT mutex.c -o mutex -lpthread
[root@localhost chapter19]# ./mutex
result : 0
```

실행결과를 보면 예제 thread4.c 에 있는 문제점이 해결되었음을 알 수 있다. 그런데 실행겨로가의 확인에는 오랜 시간이 거릴ㄴ다. 왜냐하면 뮤텍스의 lock , unlock 함수의 호출에는 생각보다 오랜시간이 걸리기 때문이다. 자! 그럼 먼저 thread_inc 함수의 동기화에 대해서 이야기 해 보자

```c
void * thread_inc (void* arg) {
    int i;
    pthread_mutex_lock(&mutex);
    for(i = 0; i < 50000000; i++) {
        num += 1;
    }
    pthread_mutex_unlock(&mutex);
    return NULL;
}
```

이는 임계영역을 상대적으로 좀 넓게 잡은 경우이다. 그런데 이는 이유없이 넓게 잡은 것이 아니고 다음의 장점을 생각한 결과이다.

"뮤텍스의 lock , unlock 함수의 호출 수를 최대한으로 제한한다."

위 예제의 thread_des 함수는 thread_inc 함수보다 뮤텍스의 lock , unlock 함수를 49,999,999 회 더 호출하는 구조이다. 때문에 인간이 느끼고도 남을 정도의 큰 속도 차이를 보인다. 따라서 쓰레드의 대기 시간이 문제가 되자 않는 상황이라면 위의 경우에는 임계영역을 넓게 잡아주는 것이 좋다. 하지만 num 의 값 증가가 50,000,000회 진행될 때까지 다른 쓰레드의 접근을 허용하지 않기 때문에 이는 단점으로 작용할 수 있다.

임계영역을 넓게 잡느냐 아니면 최대한 좁게 잡느냐는 프로그램의 성격에 따라 달리 결정할 요소이다.

## 세마포어 (Semaphore)

이번에는 세마포어를 소개하겠다. 세마포어는 뮤텍스와 매우 유사하다. 따라서 뮤텍스에서 이해한 내요을 바탕을 ㅗ쉽게 세마포어를 이해할 수 있다. 참고로 여기서는 0과 1만을 사용하는 '바이너리 세마포어'라는 것을 대상으로 쓰레드의 '실행순서 컨트롤' 중심의 동기화를 설명하겠다.

다음은 세마포어의 생성 및 소멸에 관한 함수이다.

```c
#include <semaphore.h>

int sem_init (sem_t *__sem, int __pshared, unsigned int __value)
int sem_destroy (sem_t *__sem);

// 성공 시 0, 실패시 0 이외의 값 반환

// sem : 세마포어 생성시에는 세마포어으이 참조 값 저장을 위한 변수의 주소 값 전달, 그리고 세마포어 소멸시에는 소멸하고자 하는 세마포어의 참조 값을 저장하고 있는 변수의 주소 값 전달
// pshared : 0 이외의 값 전달시, 둘 이상의 프로세스에 의해 접근 가능한 세마포어 생성, 0 전달 시 하나의 프로세스 내에서만 접근 가능한 세마포어 생성
// value : 생성되는 세마포어의 초기 값 지정

```

위의 함수에서 pshared 는 우리의 관심영역 밖이므로 0을 전달하자. 그리고 매개변수 value 에 의해 초기화되는 값은 잠시후에 알아보자. 그럼 뮤텍스의 lock , unlock 함수에 해당하는 세마포어 관련 함수를 소개하겠다.

```c
#include <semaphore.h>

int sem_wait (sem_t *__sem);
int sem_post (sem_t *__sem);

// 성공 시 0, 실패시 0 이외의 값 반환

// sem : 세마포어의 참조값을 저장하고 있는 변수의 주소 값 전달, sem_post 에 전달되면 세마모어의 값은 하나 증가, sem_wait 에 전달되면 세마포어의 값은 하나 감소
```

sem_init 함수가 호출되면 운영체제에 의해서 세마포어 오브젝트라는 것이 만들어 지는데 이곳에는 '세마포어 값(Semaphore Value)' 라는 정수가 하나 기록된다. 그리고 이 ㄱ밧은 sem_post 함수가 호출되면 1 증가하고 , sem_wait 함수가 호출되면 1 감소한다. 단! 세마포어 값은 0보다 작아질 수 없기 때문에 현재 0인 상태에서 sem_wait 함수를 호출하면, 호출한 쓰레드는 함수가 반환되지 않아서 블로킹 상태에 놓이게 된다. 물론 다른 쓰레드가 sem_post 함수를 호출하면 세마포어의 값이 1이 되므로, 이 1을 0으로 감소시키면서 블로킹 상태에서 빠져나가게 된다. 지금 설명한, 바로 이러한 특징을 이용해서 임계영역을 동기화 시키게 된다.

즉, 다음의 형태로 임계영역을 동기화 시킬 수 있다. (세마포어의 초기 값이 1이라 가정한다.)

```c
sem_wait(&sem); // 세마포어 값을 0으로
// 임계영역 시작
// ...
// 임계영역 끝
sem_post(&sem); // 세마포어 값을 1으로
```

위와 같이 코드를 구성하면 sem_wait 함수를 호출하면서 임계영역에 진입한 쓰레드가 sem_post 함수를 호출학 ㅣ전까지는 다른 쓰레드에 의해서 임계영역의 진입이 허용되지 않는다. 그리고 세마포어 값은 0과1 을 오가게 되는데, 이러한 특징때문에 위와 같은 구성을 가리켜 바이너리 세마포어라 하는 것이다.

이번에는 동시접근 동기화가 아닌 접근순서의 동기화와 관련된 예제를 작성해 보자. 예제 시나리오는 다음과 같다.

"쓰레드 A가 프로그램 사용자로부터 값ㄴ을 입력 받아서 전역변수 num에 저장을 하면, 쓰레드 B는 이 값을 가져다가 누적해 나간다. 이 과정은 총 5회 진행되고, 진행이 완료되면 총 누적금액을 출력하면서 프로그램은 종료된다"

```c
//
// Created by root on 24. 11. 10.
//
#include <stdio.h>
#include <pthread.h>
#include <semaphore.h>

void * read(void* arg);
void * accu(void * arg);
static sem_t sem_one;
static sem_t sem_two;
static int num;

int main(void) {
    pthread_t id_t1 , id_t2;
    sem_init(&sem_one , 0 , 0);
    sem_init(&sem_two , 0 , 1);

    pthread_create(&id_t1 , NULL , read , NULL);
    pthread_create(&id_t2 , NULL , accu , NULL);

    pthread_join(id_t1 , NULL);
    pthread_join(id_t2 , NULL);

    sem_destroy(&sem_one);
    sem_destroy(&sem_two);
    return 0;
}

void * read(void* arg) {
    int i;
    for(i = 0; i < 5; i++) {
        fputs("Input num : " , stdout);
        sem_wait(&sem_two);
        scanf("%d" , &num);
        sem_post(&sem_one);
    }
    return NULL;
}
void * accu(void * arg) {
    int sum = 0 , i;
    for(i = 0; i < 5; i++) {
        sem_wait(&sem_one);
        sum += num;
        sem_post(&sem_two);
    }
    printf("Result : %d" , sum);
    return NULL;
}
```

```
[root@localhost chapter19]# gcc -D_REENT_TRANT semaphore.c -o semaphore -lpthread
[root@localhost chapter19]# ./semaphore
Input num : 1
Input num : 2
Input num : 3
Input num : 4
Input num : 5
Result : 15
```
