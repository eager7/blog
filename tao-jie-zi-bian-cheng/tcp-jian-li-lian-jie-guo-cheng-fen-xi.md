# TCP建立连接过程分析

TCP在建立连接时到底哪个函数负责三路握手？ 写代码测试一下。 server：

```text
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
    while(1);
```

client：

```text
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
    printf("connect success\n");
    char aRecv[2048] = {0}; 
    const char *aSend = "This is tcp client";
    while(1)
    {
        checkError(send(iSocketFd, aSend, strlen(aSend), 0));

        printf("wait server data...\n");
        int irecv = recv(iSocketFd, aRecv, sizeof(aRecv), 0);
```

启动后查看情况：

```text
panchangtao@1003:~/WorkSpace$ netstat | grep "7878"
tcp        0      0 localhost:39709         localhost:7878          ESTABLISHED
tcp       18      0 localhost:7878          localhost:39709         ESTABLISHED
```

可以看到，server在调用了listen后就可以让客户端连上了，那么accept是做什么的呢？ 另外，7878是服务器端口，但是客户端连接上后用的却是39709作为通信端口。

那么accept是不是只是将39709返回了呢，那样的话它就只是起了个解析的作用。我们看看能否找到它的源码。

## 网上搜索，指明accept是将新建的套接字对应起来，并修改他们的状态。

今天重读《Unix网络编程》找到下面的话：  

![](../.gitbook/assets/image%20%288%29.png)

这是一段关于listen函数的解释，也就是说listen后，客户端就可以连接到服务器了，只是这时候连接的客户端都放到了一个队列中，当调用accept时从队列中取数据，如果队列为空，就阻塞。 所以客户端和服务端的三路握手是内核的TCP/IP协议驱动自己完成的，listen和accept都没有做这个工作。

