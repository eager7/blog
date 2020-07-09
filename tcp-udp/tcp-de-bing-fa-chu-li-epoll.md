# TCP的并发处理epoll

上节介绍了Socket的select并发机制，正如文中所说，select在描述符比较多时性能不够好，因为每次都要去操作所有的fd，Linux上提供了另一个高性能的并发机制，即epoll，epoll是怎么去避免select的问题的呢？ epoll有三个函数，epoll\_create，epoll\_ctl，epoll\_wait。 epoll\_create会将需要监控的fd放到集合中，不需要像select那样每次都拷贝，epoll\_ctl会为fd设置一个就绪表，然后epoll\_wait就去检测这个就绪表即可，不需要检测所有的fd，如果就绪表为空，表示没有事件，而select则需要遍历所有fd后才能知道没有事件，所以epoll的效率方面比select高。

epoll的使用方法和select类似，如果熟悉select那么使用epoll也不成问题，唯一需要注意的一点是EpollEvent.events这个参数，它表示你要监控添加fd的哪些事件以及监听方式，

```text
events可以是以下几个宏的集合：
EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
EPOLLOUT：表示对应的文件描述符可以写；
EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
EPOLLERR：表示对应的文件描述符发生错误；
EPOLLHUP：表示对应的文件描述符被挂断；
EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
```

```text
LT模式（默认）：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序可以不立即处理该事件。下次调用epoll_wait时，会再次响应应用程序并通知此事件。




ET模式：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，应用程序必须立即处理该事件。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。
ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。
```

程序示例：

epoll可以一次检测到多个描述符变化，返回值表示有多少个描述符变化了，可以轮询这个返回值获取多个数据。

```text
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
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


    int iEpollSet = epoll_create(65535);
    checkError(iEpollSet);
    struct epoll_event EpollEvent, EpollEventList[5];
    EpollEvent.data.fd = iSocketFd;
    EpollEvent.events = EPOLLIN;    //use to accept
    checkError(epoll_ctl(iEpollSet, EPOLL_CTL_ADD, iSocketFd, &EpollEvent));


    int iNumberClient = 0;
    while(1)
    {
        printf("wait epoll changed ...\n");
        int iRet = epoll_wait(iEpollSet, EpollEventList, 5, -1);
        switch(iRet){
            case (0):
                printf("select timeout\n");
            break;
            case (-1):
                printf("select error:%s\n", strerror(errno));
            break;
            default:{
                printf("get epoll events[%d]\n", iRet);
                for(i = 0; i < iRet; i++){
                    if((EpollEventList[i].events & EPOLLERR) || (EpollEventList[i].events & EPOLLHUP)){
                        printf("fd occured err:%s\n", strerror(errno));
                        continue;
                    } else if (EpollEventList[i].data.fd == iSocketFd){//server event
                        int j = 0;
                        for(j = 0; j < 5; j++){
                            if(iSockClient[j] == 0){
                                iSockClient[j] = accept(iSocketFd, (struct sockaddr*)&client_addr, &client_len);
                                checkError(iSockClient[j]);
                                printf("client connected %s\n", inet_ntoa(client_addr.sin_addr));
                                EpollEvent.data.fd = iSockClient[j];
                                EpollEvent.events = EPOLLIN | EPOLLET;/*read ,Ede-Triggered, close*/
                                checkError(epoll_ctl(iEpollSet, EPOLL_CTL_ADD, iSockClient[j], &EpollEvent));
                                iNumberClient++;
                                if(iNumberClient >= 5){
                                    checkError(epoll_ctl(iEpollSet, EPOLL_CTL_DEL, iSocketFd, &EpollEvent));
                                }
                                break;
                            }                       
                        }
                    } else {
                        int j = 0;
                        for(j = 0; j < 5; j++){
                            if((iSockClient[j] != 0) && (EpollEventList[i].data.fd == iSockClient[j])){
                                int irecv = recv(iSockClient[j], aRecv, sizeof(aRecv), 0);
                                checkError(irecv);
                                if(0 == irecv){
                                    printf("client disconnect, close it\n");
                                    checkError(epoll_ctl(iEpollSet, EPOLL_CTL_DEL, iSockClient[j], &EpollEvent));
                                    close(iSockClient[j]);
                                    iSockClient[j] = 0;
                                    iNumberClient--;
                                    if(iNumberClient < 5){
                                        EpollEvent.data.fd = iSocketFd;
                                        EpollEvent.events = EPOLLIN;
                                        checkError(epoll_ctl(iEpollSet, EPOLL_CTL_ADD, iSocketFd, &EpollEvent));
                                    }
                                } else {
                                    printf("recv data:%s\n", aRecv);
                                    checkError(send(iSockClient[j], aSend, strlen(aSend), 0));
                                }
                            }
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

