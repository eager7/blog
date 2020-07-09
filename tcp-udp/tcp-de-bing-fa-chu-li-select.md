# TCP的并发处理select

前面我们介绍的全都是一个客户端和一个服务端的情况，只适用于简单的应用场景，但是在实际的开发工作中，经常会用到一个服务器连接多个客户端的情况，如多个手机连接网关或者多个WIFI设备连接到网关等。 在Linux上处理Socket并发有好几种方式，我们着重介绍select和epoll，前者是可以跨平台的c函数，在Windows和Linux上都可以使用，后者是Linux平台特有的处理并发的函数。 本节先介绍select的使用。 select的图示：  每次调用select时，都会将描述集合fd从用户态拷贝到内核态，并为其注册回调，然后调用poll方法遍历这些fd，看看是否有描述符就绪，如果就绪了就给这个fd\_set赋值，用户根据这个fd\_set判断是哪个文件fd发生了变化，如果fd比较大，这部分开销就会变大，因为select每次都会操作所有的fd，select最大支持1024个描述符。 服务端程序如下： 有这么几个注意点： 1. select的第一个参数是监控fd set中的最大描述符加一，所以在接收到客户端请求后需要对比这个值，保持它是最大的。 2. select的监控集合fdSelect在每次调用select函数后会被清空，因此需要使用fdAll来保存fd的集合。 3. 程序中可连接的客户端最大数量为5，当达到这个值后应该将服务器的SocketFd清出fd集合，否则客户端的连接会使select不断返回，影响性能。 4. for循环后不要忘记break跳出循环，否则程序再次运行accept或者recv都会使线程挂起，select再没有机会运行。 5. 例程select中不仅做了接收客户端连接部分，并且接收了客户端的数据，但是不要在这里面处理数据，否则可能导致下次数据接收失败。 6. 多客户端通信时，可以只使用select做接收处理，然后配合非阻塞Socket进行通信，避免多线程的通信互斥问题。 7. select可以设置为非阻塞的，即设置成select\(iSelectFd + 1, &fdSelect, NULL, NULL, &tv\)模式

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
    int i = 0;
    printf("this is tcp demo\n");


    int iSocketFd = socket(AF_INET, SOCK_STREAM, 0);
    checkError(iSocketFd);


    int re = 1;
    checkError(setsockopt(iSocketFd, SOL_SOCKET, SO_REUSEADDR, &re, sizeof(re)));


    struct sockaddr_in server_addr;  
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;  
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);        /*receive any address*/
    server_addr.sin_port = htons(7878);
    checkError(bind(iSocketFd, (struct sockaddr*)&server_addr, sizeof(server_addr)));


    checkError(listen(iSocketFd, 5));


    int iSockClient[5] = {0};
    struct sockaddr_in client_addr;
    memset(&client_addr, 0, sizeof(client_addr));
    socklen_t client_len = sizeof(client_addr);


    const char *aSend = "This is tcp server";
    char aRecv[2048] = {0}; 


    fd_set fdSelect, fdAll;
    FD_ZERO(&fdAll);
    FD_SET(iSocketFd, &fdAll);
    int iSelectFd = iSocketFd;
    int iNumberClient = 0;
    while(1)
    {
        printf("wait select changed ...\n");
        fdSelect = fdAll;
        int iRet = select(iSelectFd + 1, &fdSelect, NULL, NULL, NULL);
        switch(iRet){
            case (0):
                printf("select timeout\n");
            break;
            case (-1):
                printf("select error:%s\n", strerror(errno));
            break;
            default:{
                printf("get select event \n");
                if(FD_ISSET(iSocketFd, &fdSelect)){ //accept client connect
                    for (i = 0; i < 5; ++i) {
                        if (0 == iSockClient[i]) {
                            iSockClient[i] = accept(iSocketFd, (struct sockaddr*)&client_addr, &client_len);
                            checkError(iSockClient[i]); 
                            printf("client ipaddr:%s\n", inet_ntoa(client_addr.sin_addr));
                            FD_SET(iSockClient[i], &fdAll);
                            if (iSockClient[i] > iSelectFd){
                                iSelectFd = iSockClient[i];
                            }
                            iNumberClient ++;
                            if(iNumberClient >= 5){
                                FD_CLR(iSocketFd, &fdAll);
                            }
                            break;
                        }
                    }
                } else {
                    for (i = 0; i < 5; ++i){
                        if ((iSockClient[i] != 0) && FD_ISSET(iSockClient[i], &fdSelect)){
                            int irecv = recv(iSockClient[i], aRecv, sizeof(aRecv), 0);
                            checkError(irecv);
                            if (0 == irecv){
                                printf("this client[%d] disconnect, close it\n", i);
                                FD_CLR(iSockClient[i], &fdAll);
                                close(iSockClient[i]);
                                iSockClient[i] = 0;
                                iNumberClient --;
                                if(iNumberClient < 5){
                                    FD_SET(iSocketFd, &fdAll);
                                }
                                break;
                            }
                            printf("recv client[%d] data:%s\n", i, aRecv);
                            checkError(send(iSockClient[i], aSend, strlen(aSend), 0));
                            break;
                        }       
                    }

                }
            }
            break;
        }
        sleep(0);
    }
    return 0;
}
```

