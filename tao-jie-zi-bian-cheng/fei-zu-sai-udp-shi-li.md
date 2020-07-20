# 非阻塞UDP示例

UDP编程中我们实现一个简单的C/S机制，并且实现广播和组播

2.1.1 简单的通信例子 服务端，初始化过程，设置了地址复用（可以创建多个服务器实例）和接收发送超时。

```text
int udp_server_init(char *netAddr, int port)
{
    iSocketFd = socket(AF_INET, SOCK_DGRAM, 0);//create a ucp socket file
    if(-1 == iSocketFd){
        printf("create socket fd err:%s\n", strerror(errno));
        return -1;
    }




    int on = 1;
    struct timeval timeout={2,5};//timeout
    if((-1 == setsockopt(iSocketFd,SOL_SOCKET,SO_REUSEADDR,&on,sizeof(on))) || //allow up serval server program
       (-1 == setsockopt(iSocketFd,SOL_SOCKET,SO_SNDTIMEO,&timeout,sizeof(timeout))) ||
       (-1 == setsockopt(iSocketFd,SOL_SOCKET,SO_RCVTIMEO,&timeout,sizeof(timeout)))){
        printf("setsockopt err:%s\n", strerror(errno));
        return -1;
    }




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




    if(-1 == bind(iSocketFd, (struct sockaddr*)&server_addr, sizeof(server_addr))){
        printf("bind error:%s\n", strerror(errno));
        return -1;
    }




    return 0;
}
```

INADDR\_ANY表示可以接收本机所有网络接口的数据，以及内网所有的数据，如果设成某个IP，那么表示只能接收这个IP接口发来的数据。 接收函数：（需要先建立一个新的地址变量来存放客户端地址）

```text
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
```

客户端程序：

```text
    iSocketFd = socket(AF_INET, SOCK_DGRAM, 0);//create a ucp socket file
    if(-1 == iSocketFd){
        printf("create socket fd err:%s\n", strerror(errno));
        return -1;
    }




    struct timeval timeout={2,5};//timeout
    if(-1 == setsockopt(iSocketFd,SOL_SOCKET,SO_SNDTIMEO,&timeout,sizeof(timeout)) ||
      (-1 == setsockopt(iSocketFd,SOL_SOCKET,SO_RCVTIMEO,&timeout,sizeof(timeout)))){
        printf("setsockopt err:%s\n", strerror(errno));
        return -1;
    }




    //create server socket addr struct
    struct sockaddr_in server_addr;
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_port    = 7878;
    server_addr.sin_family  = AF_INET; 
    //server_addr.sin_addr.s_addr = htonl(INADDR_ANY);//accept all interface of host
    server_addr.sin_addr.s_addr = inet_addr("127.0.0.1"); //accept one interface of host
    socklen_t server_len = sizeof(server_addr);




    char aRecv[2048] = {0};
    while(1){
        printf("wait server data...\n");
        const char *aSend = "Hello Server";
        if(-1 == sendto(iSocketFd, aSend, strlen(aSend), 0, (struct sockaddr*)&server_addr, sizeof(server_addr))){
            printf("sendto client err:%s\n", strerror(errno));
        }




        memset(aRecv, 0, sizeof(aRecv));
        if(-1 == recvfrom(iSocketFd, aRecv, sizeof(aRecv), 0, (struct sockaddr*)&server_addr, &server_len)){
            printf("recvfrom err[%d]:%s\n", errno, strerror(errno));
            if(errno == EAGAIN){
                usleep(100);
                continue;
            } else {
                exit(1);
            }
        }




        printf("server ipaddr:%s, data:%s\n", inet_ntoa(server_addr.sin_addr), aRecv);
        sleep(1);
    }
```

UDP的断开检测没有做，recvfrom的说明是返回0断开，但是实际上没有用，UDP似乎也不用关心它的状态。 此外，程序中使用了两种Socket的结构体，一种是sockaddr\_in，一种是sockaddr，这两种结构体在发送和接收时互相转换，那么为什么会有这两种结构体呢？前者的定义如下：

```text
sockaddr_in（在netinet/in.h中定义）：
struct  sockaddr_in {
short  int  sin_family;                      /* Address family */
unsigned  short  int  sin_port;              /* Port number */
struct  in_addr  sin_addr;                   /* Internet address */
unsigned  char  sin_zero[8];                 /* Same size as struct sockaddr */
};
struct  in_addr {
unsigned  long  s_addr;
};
typedef struct in_addr {
union {
      struct{
             unsigned char s_b1,
             s_b2,
             s_b3,
             s_b4;
             } S_un_b;
      struct {
             unsigned short s_w1,
             s_w2;
             } S_un_w;
      unsigned long S_addr;
      } S_un;
} IN_ADDR;
```

后者定义：

```text
struct sockaddr {
    unsigned  short  sa_family;        /* address family, AF_xxx */
    char      sa_data[14];             /* 14 bytes of protocol address */
};
```

sockaddr主要是用于各种Socket函数使用，但是在用户定义时一般不去用它，用前者会更方便。至于为什么会有两种方式，在网上找到一种解答，不知道对不对，或许这个也是因为一些历史原因导致的也说不定：

```text
网络通信中涉及到很多协议，有IPv4, IPv6, IPX, X.25, AX.25......（socket函数的domain参数表示协议族）。在网络编程中，不管用的是什么协议，使用的都是同一套socket API，而每个协议表示网络地址的结构体是不一样的，如IPv4是struct sockaddr_in，IPv6是struct sockaddr_in6，所以需要将所有这些表示网络地址的结构体抽象成一个通用的结构体，抽象之后的结构体便是struct sockaddr
```

