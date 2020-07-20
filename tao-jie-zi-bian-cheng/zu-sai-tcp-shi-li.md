# 阻塞TCP示例

TCP在默认情况下就是阻塞的方式，recv会使线程挂起，这种方式在简单通信中经常使用。

服务端代码： 服务端代码中listen的数目表示并发监听数量，并不表示最大连接数量。另外程序只是做了个简单的demo，并没有做很好的重连机制，如果发送或者接收失败，recv返回0表示连接断开，次数应该做重连。

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
    int iSockClient;
    struct sockaddr_in client_addr;
    memset(&client_addr, 0, sizeof(client_addr));
    socklen_t client_len = sizeof(client_addr);


reconnect:
    iSockClient = accept(iSocketFd, (struct sockaddr*)&client_addr, &client_len);
    checkError(iSockClient);    
    printf("client ipaddr:%s\n", inet_ntoa(client_addr.sin_addr));


    const char *aSend = "This is tcp server";
    char aRecv[2048] = {0}; 
    while(1)
    {
        printf("wait client data...\n");
        int irecv = recv(iSockClient, aRecv, sizeof(aRecv), 0);
        if(-1 == irecv){
            printf("recv err:%s\n", strerror(errno));
            if(errno == EAGAIN){
                usleep(100);
                continue;
            } else {
                exit(1);
            }
        } else if (0 == irecv){
            printf("disconnect with client\n");
            close(iSockClient);
            goto reconnect;
        }
        printf("recv client ip:%s, data:%s\n", inet_ntoa(client_addr.sin_addr), aRecv);
        checkError(send(iSockClient, aSend, strlen(aSend), 0));
        sleep(1);
    }
    return 0;
}
```

客户端代码： 客户端代码中并不需要bind，它的重连需要重新建立Socket描述字，因为TCP是双向通信，服务器断开后并没有将整个Socket断开，它只是处于半连接状态，按照《TCP/IP协议》的说法，此时还可以向服务端发送数据，但是服务端程序因为没有去监控此Socket，所以数据全部会丢失，想重连服务器就得将Socket关掉重建。

```text
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <errno.h>
#include <arpa/inet.h>
#include <unistd.h> //usleep
#include <signal.h>
#define checkError(ret) do{if(-1==ret){printf("[%d]err:%s\n", __LINE__, strerror(errno));exit(1);}}while(0)
int main(int argc, char const *argv[])
{
    printf("this is a tcp client demo\n");
    signal(SIGPIPE, SIG_IGN);//ingnore signal interference
    int iSocketFd = 0;
reconnect: 
    iSocketFd = socket(AF_INET, SOCK_STREAM, 0);
    checkError(iSocketFd);


    int re = 1;
    checkError(setsockopt(iSocketFd, SOL_SOCKET, SO_REUSEADDR, &re, sizeof(re)));


    struct sockaddr_in server_addr;  
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;  
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);        /*receive any address*/
    //server_addr.sin_addr.s_addr = inet_addr("10.128.118.43");
    server_addr.sin_port = htons(7878);
    while(1){
        printf("conncet to server...\n");
        if(-1 != connect(iSocketFd, (struct sockaddr *)&server_addr, sizeof(server_addr))){
            break;
        }   
        sleep(1);
    }


    char aRecv[2048] = {0}; 
    const char *aSend = "This is tcp client";
    while(1)
    {
        checkError(send(iSocketFd, aSend, strlen(aSend), 0));

        printf("wait server data...\n");
        int irecv = recv(iSocketFd, aRecv, sizeof(aRecv), 0);
        if(-1 == irecv){
            printf("recv err[%d]:%s\n", errno, strerror(errno));
            if(errno == EAGAIN){
                usleep(100);
                continue;
            } else {
                exit(1);
            }
        } else if (0 == irecv){
            printf("disconnect with server\n");
            close(iSocketFd);
            goto reconnect;
        }
        printf("server ip:%s, data:%s\n", inet_ntoa(server_addr.sin_addr), aRecv);
        sleep(1);
    }
    return 0;
}
```

================================================================== 上面代码运行起来应该是什么样的结果呢？下面是在同一台机器上运行的结果：

客户端：

```text
this is a tcp client demo
wait server data...
server ip:0.0.0.0, data:This is tcp server
```

服务端：

```text
this is tcp demo
client ipaddr:127.0.0.1
wait client data...
recv client ip:127.0.0.1, data:This is tcp client
```

注意到IP了么，Linux的网络Socket并没有出本机，或者说它可能没有走TCP/IP的标准流程，直接在内部做了转发，所以在本机通信，可以不必用本地Socket，直接用网络Socket即可。 但是这个特性需要将Socket的地址设成INADDR\_ANY，如果改成下面形式就会走IP层了：

```text
    //server_addr.sin_addr.s_addr = htonl(INADDR_ANY);        /*receive any address*/
    server_addr.sin_addr.s_addr = inet_addr("10.128.118.43");
```

输出如下：

```text
this is a tcp client demo
wait server data...
server ip:10.128.118.43, data:This is tcp server
this is tcp demo
client ipaddr:10.128.118.43
wait client data...
recv client ip:10.128.118.43, data:This is tcp client
```

