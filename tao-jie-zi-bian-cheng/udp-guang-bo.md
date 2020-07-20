# UDP广播

\[TOC\]

上一节我们介绍了UDP的简单示例，下面介绍udp的广播使用。

广播Socket需要设置SO\_BROADCAST，后面的参数表示是否有广播权限，如果参数为0，就无法进行广播，因为系统会禁止程序使用广播权限。 另外，地址INADDR\_BROADCAST的值其实就是255.255.255.255。

## 客户端程序：

```text
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <arpa/inet.h>
#include <unistd.h> //usleep
#define checkError(ret) do{if(-1==ret){printf("[%d]err:%s\n", __LINE__, strerror(errno));exit(1);}}while(0)
int main(int argc, char const *argv[])
{
    printf("udp broadcast client\n");
    int iSocketFd = socket(AF_INET, SOCK_DGRAM, 0);
    checkError(iSocketFd);
    int on = 1;
    int broadcastEnable = 1;//the permissions of broadcast
    checkError(setsockopt(iSocketFd, SOL_SOCKET, SO_BROADCAST, (char *)&broadcastEnable, sizeof(broadcastEnable)));
    checkError(setsockopt(iSocketFd,SOL_SOCKET,SO_REUSEADDR,&on,sizeof(on)));
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family     = AF_INET; 
    server_addr.sin_port     = 7878;
    server_addr.sin_addr.s_addr = htonl(INADDR_BROADCAST);
    char aRecv[2048] = {0};
    const char *aSend = "This is udp brocast client";    
    while(1)
    {
        printf("send hello to server\n");
        checkError(sendto(iSocketFd, aSend, strlen(aSend), 0, (struct sockaddr*)&server_addr, sizeof(server_addr)));
        printf("wait server data...\n");
        memset(aRecv, 0, sizeof(aRecv));
        int irecv = recv(iSocketFd, aRecv, sizeof(aRecv), 0);
        if(-1 == irecv){
            printf("recvfrom err:%s\n", strerror(errno));
            if(errno == EAGAIN){
                usleep(100);
                continue;
            } else {
                exit(1);
            }
        }
        printf("recv server data:%s\n", aRecv);
        if(!strcmp(aRecv, "This is udp broadcast server")){
            break;
        }
        sleep(1);
    }

    return 0;
}
```

## 服务端程序

就是个普通的udp服务，收到广播数据后返回给发送数据的客户端即可：

```text
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <arpa/inet.h>
#include <unistd.h> //usleep
#define checkError(ret) do{if(-1==ret){printf("[%d]err:%s\n", __LINE__, strerror(errno));exit(1);}}while(0)
int main(int argc, char *argv[])
{
    printf("udp server simple demo\n");
    int iSocketFd = socket(AF_INET, SOCK_DGRAM, 0);//create a ucp socket file
    checkError(iSocketFd);
    int on = 1;
    int broadcastEnable = 1;
    checkError(setsockopt(iSocketFd,SOL_SOCKET,SO_REUSEADDR,&on,sizeof(on)));
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family     = AF_INET; 
    server_addr.sin_port     = 7878;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);//accept all interface of host
    checkError(bind(iSocketFd, (struct sockaddr*)&server_addr, sizeof(server_addr)));
    //create a client addr
    struct sockaddr_in client_addr;
    memset(&client_addr, 0, sizeof(client_addr));
    socklen_t client_len = sizeof(client_addr);

    const char *aSend = "This is udp broadcast server";
    char aRecv[2048] = {0};
    while(1){
        printf("wait client data...\n");
        memset(aRecv, 0, sizeof(aRecv));
        int irecv = recvfrom(iSocketFd, aRecv, sizeof(aRecv), 0, (struct sockaddr*)&client_addr, &client_len);
        if(-1 == irecv){
            printf("recvfrom err[%d]:%s\n", errno, strerror(errno));
            if(errno == EAGAIN){
                usleep(100);
                continue;
            } else {
                exit(1);
            }
        }
        printf("client ipaddr:%s, data:%s\n", inet_ntoa(client_addr.sin_addr), aRecv);
        int isend = sendto(iSocketFd, aSend, strlen(aSend), 0, (struct sockaddr*)&client_addr, sizeof(client_addr));
        if(-1 == isend){
            printf("sendto client err:%s\n", strerror(errno));
        }
        sleep(1);
    }
    return 0;
}
```

