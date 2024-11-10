## 클라이언트 소켓의 구현

앞서 보인 서버 프로그램에서 생성한 소켓을 가리켜 '서버 소켓' 또는 '리스닝(listening) 소켓'이라고 한다. 반면에 이번에 소개할 소켓은 연결 요청을 진행하는 '클라이언트 소켓'이다. 참고로 클라이언트 소켓의 생성과정은 서버 소켓의 생성과정에 비해 상대적으로 간단하다.
<br/> 

앞에서는 전화를 거는 (연결을 요청하는)기능의 함수를 소개하지 않았다. 이는 클라이언트 소켓을 대상으로 호ㅗ출하는 함수이기 때문이다. 다음은 전화를 거는 기능의 함수이다.
```c
#include <sys/socket.h>

int connect(int sockfd, struct sockaddr *serv_addr , socklen_t addrlen); // 성공시 0 , 실패시 -1
```

클라이언트 프로그램에서  socket 함수호출을 통한 소켓의 생성과 connect 함수 호출을 통한 서버의 연결 요청과정만이 존재한다. 때문에 서버 프로그램에 비해 간단하다.

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
    int sock;
    struct sockaddr_in serv_addr;
    char message[30];
    int str_len;

    if (argc != 3)
    {
        printf("Usage : %s <IP> <Port> \n", argv[0]);
        exit(1);
    }

    sock = socket(PF_INET, SOCK_STREAM, 0);

    if (sock == -1)
        error_handling("socket() error");

    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = inet_addr(argv[1]);
    serv_addr.sin_port = htons(atoi(argv[2]));

    if (connect(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) == -1)
        error_handling("connect() error");

    str_len = read(sock, message, sizeof(message) - 1);
    if (str_len == -1)
        error_handling("read() error");

    message[str_len] = '\0';  // Null terminate the received message

    printf("Message from server: %s\n", message);
    close(sock);

    return 0;
}

void error_handling(char *message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}

```

