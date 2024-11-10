## Iterative 기반의 서버 , 클라이언트 구현
마지막으로 에코 서버와 에코 클라이언트를 구현해보자. 에코 서버는 클라이언트가 전송하는 문자열 데이터를 그대로 재전송하는 , 말 그대로 문자열 데이터를 에코(echo) 시키는 서버이다. 그런데 이의 구현에 앞서 먼저 Iterative 서버의 구현에 대해 알아보자

## Iterative 서버의 구현
지금까지 우리가 보아온 Hello World 서버는 한 클라이언트의 요청에만 응답을 하고 바로 종료되어버렸다. 때문에 연결요청 대기 큐의 크기도 사실상 의미가 없었다 그런데 이는 우리가 생각해 오던 서버의 모습이 아니다. 큐의 크기까지 설정해 놓았다면 연결 요청을 하는 모든 클라이언트에게 약속되어 있는 서비스를 제공해야한다. 그렇다면 계속해서 들어오는 클라이언트의 연결요청을 수락하기 위해서는 서버의 코드 구현을 어떠한 식으로 확장해야 할까? 

단순히 반복문을 삽입해서 accept 함수를 반복호출하면 된다.

1. socket() : 소켓생성
2. bind() : 소켓 주소 할당
3. listen() : 연결요청 대기 상태

-- while <br/>
4. accept() : 연결 허용 <br/>
5. read() / write() : 데이터 송수신 <br/>
6. close(client) : 클라이언트 연결 종료 <br/>
--

7. close(server) : 서버 연결 종료

"그럼 뭐예요! 은행창구도 아니고, 그래도 명색이 서버인데 한 순간에 하나의 클라이언트에게만 서비스를 제공할 수 있다는 거예요?"

그렇다! 명색이 서버인데 한 순간에 하나의 클라이언트에게만 서비스를 제공할 수 있다. 그러나 이후에 프로세스와 쓰레드에 대해 공부하고 나면 동시에 둘 이상의 클라이언트에게 서비스를 제공하는 서비스를 만들 수 있게 된다.

## Iterative 에코 서버 , 에코 클라이언트
앞서 설명한 형태의 서버를 가리켜 Iterative 서버라고 한다. 그리고 서버가 Iterative 형태로 동작한다 해도 클라이언트 코드에는 차이가 없을을 이해할 수 있다. 그럼 코드를 작성해보자 먼저 프로그램의 기본 동작방식을 정리해보자

1. 서버는 한 순간에 하나의 클라이언트와 연결되어 에코 서비스를 제공한다.
2. 서버는 총 다섯개의 클라이언트에게 순차적으로 서비스를 제공하고 종료한다.
3. 클라이언트는 프로그램 사용자로부터 문자열 데이터를 입력받아서 서버에 전송한다.
4. 서버는 전송받은 문자열 데이터를 클아이언트에게 재전송한다. 즉, 에코 시킨다.
5. 서버와 클라이언트간의 문자열 에코는 클라이언트가 Q를 입력할 때 까지 계속한다.


## 서버 코드
```c
#include <sys/socket.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>


#define BUF_SIZE 1024
void errorHandling(char* message);

int main (int argc , char *argv[]) {

        int servSock , clientSock;
        char message[BUF_SIZE];
        int strLen , i;

        struct sockaddr_in servAddr , clientAddr;
        socklen_t clientAddrSz;

        if(argc != 2) {
                printf("Usage : %s <port> \n" , argv[0]);
                exit(1);
        }

        servSock = socket(PF_INET, SOCK_STREAM, 0);
        if(servSock == -1) {
                errorHandling("socket() error!");
        }

        // 서버 addr  초기화
        memset(&servAddr , 0  , sizeof(servAddr));
        servAddr.sin_family = AF_INET;
        servAddr.sin_addr.s_addr = htonl(INADDR_ANY);
        servAddr.sin_port = htons(atoi(argv[1]));

        if(bind(servSock , (struct sockaddr*) &servAddr , sizeof(servAddr)) == -1)
                errorHandling("bind() error!");


        if(listen(servSock , 5) == -1) {
                errorHandling("listen() error!");
        }


        clientAddrSz = sizeof(clientAddr);

        for(i = 0; i < 5;i++) {
                clientSock = accept(servSock , (struct sockaddr*) &clientAddr , &clientAddrSz);

                if(clientSock == -1)
                        errorHandling("accpet() ERROR!");
                else
                        printf("connected client %d \n" , i + 1);

                while((strLen = read(clientSock , message , BUF_SIZE)) != 0) {
                        printf("Message from client : %s" , message);
                        write(clientSock , message, strLen);
                }

                close(clientSock);
        }
        close(servSock);
        return 0;
}

void errorHandling(char* message) {
        fputs(message , stderr);
        fputc('\n' , stderr);
        exit(1);
}
```

## 클라이언트 코드
```c
#include <sys/socket.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define BUF_SIZE 1024
void errorHandling(char* message);

int main(int argc , char* argv[]) {

        if(argc != 3) {
                printf("Usage : %s <IP> <PORT> \n" , argv[0]);
        }

        int sock;
        char message[BUF_SIZE];
        int strLen;
        struct sockaddr_in serv_addr;

        sock = socket(PF_INET , SOCK_STREAM , 0);

        if(sock == -1)
                errorHandling("socket() error!");


        memset(&serv_addr , 0 , sizeof(serv_addr));
        serv_addr.sin_family = AF_INET;
        serv_addr.sin_addr.s_addr = inet_addr(argv[1]);
        serv_addr.sin_port = htons(atoi(argv[2]));

        if(connect(sock , (struct sockaddr*) &serv_addr , sizeof(serv_addr)) == -1) {
                errorHandling("connect () error!");
        } else
                puts("connected .....");

        while(1) {

                fputs("Input message (Q to quit) : " , stdout);
                fgets(message , BUF_SIZE , stdin);

                if(strcmp(message , "q\n") == 0 || strcmp(message , "Q\n") == 0) {
                        break;
                }

                write(sock , message , strlen(message));
                strLen = read(sock , message , BUF_SIZE - 1);
                message[strLen] = 0;
                printf("Message from server : %s \n" , message);
        }

        close(sock);
        return 0;
}


void errorHandling(char* message) {
        fputs(message , stderr);
        fputc('\n' , stderr);
        exit(1);
}
```

```
// 서버
[centos@localhost code2]$ ./echo_server 9190
connected client 1
Message from client : hi

// 클라이언트

[root@localhost code2]# ./echo_client 127.0.0.1 9190
connected .....
Input message (Q to quit) : hi
Message from server : hi

Input message (Q to quit) : q
```

<br/>
<br/>

## 에코 클라이언트의 문제점
다음은 45 ~ 48 행에 삽입된 입출력 문장이다.
```c
write(sock , message , strlen(message));
strLen = read(sock , message , BUF_SIZE - 1);
message[strLen] = 0;
printf("Message from server : %s \n" , message);
```

위의 코드는 다음과 같은 잘못된 가정이 존재한다.

"read ,write 함수가 호출될 때마다 문자열 단위로 실제 입출력이 이뤄진다."

문론 write 함수를 호출할 때마다 하나의 문장을 전송하니 이렇게 가정하는 것도 무리는 아니다. 하지만 TCP 는 데이터의 경계가 존재하지 않는다고 했던 Chapter 02 의 설명을 기억하는가? 위에서 구현한 클라이언트는 TCP 클라이언트이기 때문에 둘 이상의 write 함수 호출로 전달된 문자열 정보가 묶여서 한번에 서버로 전송될 수 ㅣㅇㅆ다. 그리고 그러한 상황이 발생하면 클라이언트는 한번에 둘 이상의 문자열 정보를 서버로부터 되돌려 받아서 원하는 결과를 어지 못할 수 있다. 그리고 서버가 다음과 같이 판단하는 상황도 생각해봐야한다.

"문자열의 길이가 제법 긴 편이니, 문자열을 두 개의 패킷에 나눠서 보내야겠군!"

서버는 한번의 write 함수호출로 데이터 전송을 명령했지만, 전송할 데이터의 크기가 크다면 운영체제는 내부적으로 이를 여러개의 조각으로 나눠서 클라이언트에게 전송할 수도 있는 일이다. 그리고 이 과정에서 데이터의 모든 조각이 클라이언트에게 전송되지 않았음에도 불구하고 클라이언트는 read 함수를 호출할 지도 모른다. 이 모든 문제가 TCP 의 데이터 전송 특성에서 비롯된 것이다. 그렇다면 이 문제를 어떻게 해결해야 할까? 다음 chpater 에서 알아보자


다음 chpater 5장