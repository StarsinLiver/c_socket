## TCP의 입출력 루틴(Routine) 분할
fork 와 관련된 매우 의미있는 내용 전부를 이해했다. 그래서 내친김에 이를 바탕으로 '입출력 루틴의 분할'이라는 것을 클라이언트 영역에서 시도해보자 

## 입출력 루틴 불할의 의미와 이점
지금까지는 한번 데이터를 전송하면 에코되어 돌아오는 데이터를 수신할 때까지 마냥 기다려야 했다. 이유가 무엇인가? 프로그램 코드의 흐름이 read 와 write 를 반복하는 구조였기 때문이다. 그런데 이렇게 밖에 구현할 수 없었던 이유는 하나의 프로세스를 기반으로 프로그램이 동작했기 때문이다. 그러나 이제는 둘 이상의 프로세스를 생성할 수 있으니, 이를 바탕으로 데이터의 송신과 수신을 분리해보자

![alt text](/image/image10.png)

클라이언트의 부모 프로세스는 데이터의 수신을 담당하고, 별도로 생성된 자식 프로세스는 데이터의 송신을 담당한다. 그리고 이렇게 구현해 놓으면 입력과 출력을 담당하는 프로세스가 각각 다르기 대문에 서버로부터의 데이터 수신여부에 상관없이 데이터를 전송할 수 있다.

## 에코 클라이언트의 입출력 루틴 분할
```c
#include <sys/socket.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>


#define BUF_SIZE 30
void error_handling(char * message);
void read_routine(int sock , char *buf);
void write_routine(int sock , char *buf);

int main(int argc, char const *argv[])
{
  int sock;
  struct sockaddr_in sockaddr;
  char buf[BUF_SIZE];
  pid_t pid;

  if(argc != 3) {
    printf("Usage : %s <IP> <port> \n" , argv[0]);
    exit(1);
  }

  sock = socket(PF_INET , SOCK_STREAM , 0);
  memset(&sockaddr , 0 , sizeof(sockaddr));
  sockaddr.sin_family = AF_INET;
  sockaddr.sin_addr.s_addr = inet_addr(argv[1]);
  sockaddr.sin_port = htons(atoi(argv[2]));

  if(connect(sock ,(struct sockaddr*) &sockaddr , sizeof(sockaddr)) == -1)
    error_handling("connect() error!");
  
  pid = fork();

  if(pid == 0)
    write_routine(sock , buf); // 자식 프로세스는 쓰기 진행
  else  
    read_routine(sock , buf); // 부모 프로세스는 읽기 진행

  close(sock);
  return 0;
}

void read_routine(int sock , char* buf) {
  while(1) {
    int str_len = read(sock , buf , BUF_SIZE);
    if(str_len == 0)
      return;
    
    buf[str_len] = 0;
    printf("Message from server %s \n" , buf);
  }
}

void write_routine(int sock , char* buf) {
  while(1) {
    fgets(buf , BUF_SIZE , stdin);

    if(!strcmp(buf , "q\n") || !strcmp(buf , "Q\n")){
      shutdown(sock , SHUT_WR);
      return ;
    }

    write(sock , buf , strlen(buf));
  }
}

void error_handling(char * message) {
  fputs(message , stderr);
  fputc('\n' , stderr);
  exit(1);
}
```

```
dsa
Message from server dsa
 
hello
Message from server hello
 
my name
Message from server my name
```

