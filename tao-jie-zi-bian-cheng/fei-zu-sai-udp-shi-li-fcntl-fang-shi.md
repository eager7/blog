# 非阻塞UDP示例\(fcntl方式\)

在设置Socket的非阻塞模式时，除了使用setsockopt外，还可以使用fcntl的方式来设置，下面是例程。 需要注意，用fcntl的方式不能设置超时时间，recvfrom函数会在读不到数据后马上返回，需要手动做延时，和setsockopt的应用场景不同，用户需要根据需求自行选取。

```text
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <arpa/inet.h>
#include <unistd.h> //usleep
#include <fcntl.h>


#define checkError(ret) do{if(-1==ret){printf("[%d]err:%s\n", __LINE__, strerror(errno));exit(1);}}while(0)
int udp_server_init(char *netAddr, int port);


int iSocketFd = 0;


int main(int argc, char *argv[])
{
    printf("udp server simple demo\n");
    if(0 != udp_server_init(NULL, 7878)){
        printf("udp_server_init error\n");
        exit(1);
    }


    //create a client addr
    struct sockaddr_in client_addr;
    memset(&client_addr, 0, sizeof(client_addr));
    socklen_t client_len = sizeof(client_addr);


    char aRecv[2048] = {0};
    while(1){
        printf("wait client data...\n");
        memset(aRecv, 0, sizeof(aRecv));
        int irecv = recvfrom(iSocketFd, aRecv, sizeof(aRecv), 0, (struct sockaddr*)&client_addr, &client_len);
        if(-1 == irecv){
            printf("recvfrom err:%s\n", strerror(errno));
            if(errno == EAGAIN){
                usleep(100);
                continue;
            } else {
                exit(1);
            }
        }


        printf("client ipaddr:%s, data:%s\n", inet_ntoa(client_addr.sin_addr), aRecv);
        const char *aSend = "This is udp server";
        int isend = sendto(iSocketFd, aSend, strlen(aSend), 0, (struct sockaddr*)&client_addr, sizeof(client_addr));
        if(-1 == isend){
            printf("sendto client err:%s\n", strerror(errno));
        }
        sleep(1);
    }
    return 0;
}


int udp_server_init(char *netAddr, int port)
{
    iSocketFd = socket(AF_INET, SOCK_DGRAM, 0);//create a ucp socket file
    checkError(iSocketFd);
    int on = 1;
    checkError(setsockopt(iSocketFd,SOL_SOCKET,SO_REUSEADDR,&on,sizeof(on)));
    checkError(fcntl(iSocketFd, F_SETFL, (fcntl(iSocketFd, F_GETFL) | O_NONBLOCK)));//set noblock
    //create socket addr struct
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family  = AF_INET; 
    server_addr.sin_port    = port;
    if(NULL == netAddr){
        server_addr.sin_addr.s_addr = htonl(INADDR_ANY);//accept all interface of host
    } else {
        server_addr.sin_addr.s_addr = inet_addr(netAddr); //accept one interface of host
    }
    checkError(bind(iSocketFd, (struct sockaddr*)&server_addr, sizeof(server_addr)));
    return 0;
}
```

