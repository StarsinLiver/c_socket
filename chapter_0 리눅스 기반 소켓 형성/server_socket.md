## 서버 소켓을 생성을 해보자

전화기를 생각을 해보자 우선 집에 전화를 놓으려면 먼저 전화기를 하나 구입해야 한다. 그럼 전화기를 하나 장만해보자
<br/>
다음은 소켓을 생성하는 함수이다.

```c
#include <sys/socket.h>

int socket(int domain , int type , int protocol)
```

다음은 전화번호를 부여 받아야한다. 다음 함수를 통해서 앞서 생성한 소켓에  IP 와 포트번호라는 소켓의 주소정보에 해당하는 것을 할당해보자

```c
#include <sys/socket.h>
int bind(int sockfd , struct sockaddr *myaddr , socklen_t addrlen);
```

전화기가 전화 케이블에 연결하는 순간 전화를 받을 수 있게 되는데 소켓도 마찬가지로 연결요청이 가능한 상태가 되어야 된다.
다음 함수는 소켓을 연결요청이 가능한 상태가 되게 한다.

```c
#include <sys/socket.h>

int listen(int sockfd , int backlog); // 성공시 0 실패시 -1 반환
```

이제 전화기가 전화 케이블에 연결되었으니 전화벨이 울릴것이고, 전화벨이 울리면 통화를 위해서 수화기를 들어야 한다.
<br/>
수화기를 들었다는 것은 연결 요청에 대한 수락을 의마한다.
누군가 데이터의 송수신을 위해 연결요청을 해오면 다음 함수호출을 통해서 그 요청을 수락해야 한다.

```c
#include <sys/socket.h>
int accept (int sockfd , struct sockaddr *addr , socklen_t *addrlen)
```

위의 과정을 다음과 같이 정리할 수 있다.
<br/>

1. 소켓생성                 [ socket 함수호출 ]
2. IP 주소와 PORT 번호 할당 [ bind 함수호출 ]
3. 연결요청 가능상태로 변경 [ listen 함수호출 ]
4. 연결요청에 대한 수락     [ accep 함수호출 ]


<br/>
<br/>

## "Hello world" 서버 프로그램 구현
앞서 설명한 함수의 호출과 정을 확인하기 위해서 연결요청 수락 시 "Hello world"라고 응답해주는 서버 프로그램을 작성해보자

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

void error_handling(char *message);

int main(int argc, char *argv[])
{
    int serv_sock;
    int clnt_sock;
    struct sockaddr_in serv_addr;
    struct sockaddr_in clnt_addr;
    socklen_t clnt_addr_size;

    char message[] = "Hello world";

    if (argc != 2)
    {
        printf("Usage : %s <port>\n", argv[0]);
        exit(1);
    }

    serv_sock = socket(PF_INET, SOCK_STREAM, 0);
    if (serv_sock == -1)
        error_handling("socket() error");

    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_addr.sin_port = htons(atoi(argv[1]));

    if (bind(serv_sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) == -1)
        error_handling("bind() error");

    if (listen(serv_sock, 5) == -1)
        error_handling("listen() error");

    clnt_addr_size = sizeof(clnt_addr);
    clnt_sock = accept(serv_sock, (struct sockaddr *)&clnt_addr, &clnt_addr_size);
    if (clnt_sock == -1)
        error_handling("accept() error");

    write(clnt_sock, message, sizeof(message));
    close(clnt_sock);
    close(serv_sock);

    return 0;
}

void error_handling(char *message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}

```

