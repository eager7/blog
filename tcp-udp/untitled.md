# 局域网发现协议

﻿局域网发现协议

网关和手机之间的发现协议应该用一个可以自动匹配IP的方式。

比如说，网关刚开始是处于路由模式的，那么它的IP地址可能是192.168.1.1，当用户将它切换成STA模式或者说桥接模式时，它的IP地址一定会变化，那么这个时候手机就无法获取设备的IP了。

网上经常说用组播去做发现协议，但是我经过测试后发现，组播只适用于一般的场景，比如手机只会连接网关，或者说只会去连接它的上级路由。因为组播协议在路由端会遇到路由的不同处理，有的可能无法出局域网导致无法发现。

组播想发送到自己的子网需要添加下面一条路由 协议：

```text
route add -net 224.0.0.0 netmask 224.0.0.0 dev br-lan
```

此路由协议后面的接口改一下的话同样适用于上级路由。

组播的代码如下：

初始化

```text
if(-1 == (sIotcMulticast.iSocketFd = socket(AF_INET, SOCK_DGRAM, 0)))
    {
        ERR_vPrintf(T_TRUE, "socket create error %s\n", strerror(errno));
        return E_MULTI_ERROR_CREATESOCK;
    }
    
    int on = 1;  /*SO_REUSEADDR port can used twice by program */
    if((setsockopt(sIotcMulticast.iSocketFd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)))<0) 
    {  
        ERR_vPrintf(T_TRUE,"setsockopt failed, %s\n", strerror(errno));  
        close(sIotcMulticast.iSocketFd);
        return E_MULTI_ERROR_SETSOCK;
    }  


    sIotcMulticast.server_addr.sin_family = AF_INET;  
    sIotcMulticast.server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    sIotcMulticast.server_addr.sin_port = htons(iPort);
    if(-1 == bind(sIotcMulticast.iSocketFd, 
                    (struct sockaddr*)&sIotcMulticast.server_addr, sizeof(sIotcMulticast.server_addr)))
    {
        ERR_vPrintf(T_TRUE,"bind socket failed, %s\n", strerror(errno));  
        close(sIotcMulticast.iSocketFd);
        return E_MULTI_ERROR_BIND;
    }


    sIotcMulticast.multi_addr.imr_multiaddr.s_addr = inet_addr(paMulAddress);
    sIotcMulticast.multi_addr.imr_interface.s_addr = htonl(INADDR_ANY);
    if(setsockopt(sIotcMulticast.iSocketFd, IPPROTO_IP, IP_ADD_MEMBERSHIP, 
            (char *)&sIotcMulticast.multi_addr, sizeof(sIotcMulticast.multi_addr)) < 0)
    {
        ERR_vPrintf(T_TRUE,"setsockopt failed, %s\n", strerror(errno));  
        close(sIotcMulticast.iSocketFd);
        return E_MULTI_ERROR_SETSOCK;
    }
```

然后起一个线程作为服务器，等待手机发送请求，收到请求后随便回复一个什么值，手机会根据这个包去解析出IP，无需显式的发送：

```text
while(sIotcMulticast.eThreadState)
    {
        sched_yield();
        if((iRecvLen = recvfrom(sIotcMulticast.iSocketFd, paRecvBuffer, MDBF, 0, 
                    (struct sockaddr*)&sIotcMulticast.server_addr,(socklen_t*)&iAddrLen)) < 0)
        {
            ERR_vPrintf(T_TRUE, "Recvfrom Data Error!\n");
        }
        BLUE_vPrintf(DBG_MUL, "Recv Data: %s\n", paRecvBuffer);
        const char *paResponse = "This is Server";
        if(sendto(sIotcMulticast.iSocketFd, paResponse, strlen(paResponse), 0, 
                    (struct sockaddr*)&sIotcMulticast.server_addr, sizeof(sIotcMulticast.server_addr)) < 0)
        {
            ERR_vPrintf(T_TRUE, "Send Data Error!\n");
        }
        
        sleep(0);
    }
```

手机端代码：

```text
class SocketSearchThread extends Thread{
        public String stringSearch;


        public SocketSearchThread(String str){
            stringSearch = str;
        }


        @Override
        public void run() {
            Message msgSocket = new Message();
            utils.DBG_vPrintf("Create Mul Socket");
            try {
                MulticastSocket mSocket = new MulticastSocket(6789);
                InetAddress groupAddress = InetAddress.getByName(Utils.stringMulAddress);
                mSocket.joinGroup(groupAddress);
                mSocket.setTimeToLive(4);


                utils.DBG_vPrintf("Send Data to Server");
                byte[] buffSearch = new byte[255];
                buffSearch = stringSearch.getBytes("utf-8");
                DatagramPacket udpPacket = new DatagramPacket(buffSearch, buffSearch.length, groupAddress, Utils.iMulPort);
                mSocket.send(udpPacket);


                byte[] byteRev = new byte[512];
                udpPacket = new DatagramPacket(byteRev,byteRev.length);
                utils.DBG_vPrintf("Recv Data From Server");
                mSocket.setSoTimeout(2000);//Set Recv Timeout
                mSocket.receive(udpPacket);


                String stringIotcAddress;
                stringIotcAddress = new String(udpPacket.getData()).trim();
                utils.DBG_vPrintf("Recv Data From Server Success:" + stringIotcAddress);


                Message msgIotcAddress = new Message();
                msgIotcAddress.what = Utils.iFindServer;
                msgIotcAddress.obj = udpPacket.getAddress().getHostAddress();
                utils.DBG_vPrintf("The Server Ip is " + msgIotcAddress.obj.toString());


                handlerSocketRev.sendMessage(msgIotcAddress);


                mSocket.leaveGroup(groupAddress);
                mSocket.disconnect();
                mSocket.close();
            } catch (IOException e) {
                utils.ERR_vPrintf("Can't Search Iotc," + e.toString());
                e.printStackTrace();
                handlerSocketRev.sendEmptyMessage(Utils.iTimeOut);
            }


        }
    }
```

================================================================================================

上面的组播经测试发现并不好用，网段一变就无法连接，下面介绍广播的方式。

广播的客户端代码：

```text
static teBroadStatus IotcBroadcastSocketInit(int iPort, char *paNetAddress)
{
    DBG_vPrintf(DBG_BROAD, "IotcBroadcastSocketInit\n");
    if(NULL == paNetAddress)
    {
        ERR_vPrintf(T_TRUE, "The Param is Error\n");
        return E_BROAD_ERROR_PARAM;
    }
    
    if(-1 == (sIotcBroadcast.iSocketFd = socket(AF_INET, SOCK_DGRAM, 0)))
    {
        ERR_vPrintf(T_TRUE, "socket create error %s\n", strerror(errno));
        return E_BROAD_ERROR_CREATESOCK;
    }
    
    int on = 1;  /*SO_REUSEADDR port can used twice by program */
    if((setsockopt(sIotcBroadcast.iSocketFd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)))<0) 
    {  
        ERR_vPrintf(T_TRUE,"setsockopt failed, %s\n", strerror(errno));  
        close(sIotcBroadcast.iSocketFd);
        return E_BROAD_ERROR_SETSOCK;
    }  


    bzero(&sIotcBroadcast.server_addr,sizeof(sIotcBroadcast.server_addr));  
    sIotcBroadcast.server_addr.sin_family=AF_INET;  
    sIotcBroadcast.server_addr.sin_addr.s_addr=htonl(INADDR_ANY);  
    sIotcBroadcast.server_addr.sin_port=htons(iPort);  
    
    if(-1 == bind(sIotcBroadcast.iSocketFd, 
                    (struct sockaddr*)&sIotcBroadcast.server_addr, sizeof(sIotcBroadcast.server_addr)))
    {
        ERR_vPrintf(T_TRUE,"bind socket failed, %s\n", strerror(errno));  
        close(sIotcBroadcast.iSocketFd);
        return E_BROAD_ERROR_BIND;
    }


    BLUE_vPrintf(DBG_BROAD, "Create Socket Fd %d\n", sIotcBroadcast.iSocketFd);
    return E_BROAD_OK;
}
```

线程体：

```text
while(sIotcBroadcast.eThreadState)
    {
        sched_yield();
        if((iRecvLen = recvfrom(sIotcBroadcast.iSocketFd, paRecvBuffer, MDBF, 0, 
                    (struct sockaddr*)&sIotcBroadcast.server_addr,(socklen_t*)&iAddrLen)) < 0)
        {
            ERR_vPrintf(T_TRUE, "Recvfrom Data Error!%s\n", strerror(errno));
            usleep(5);
            continue;
        }
        struct sockaddr_in *p = (struct sockaddr_in*)&sIotcBroadcast.server_addr;
        BLUE_vPrintf(DBG_BROAD, "Recv Data[%d]: %s, from %s\n", iRecvLen, paRecvBuffer, inet_ntoa(p->sin_addr));
        const char *paResponse = "This is Server";
        if(sendto(sIotcBroadcast.iSocketFd, paResponse, strlen(paResponse), 0, 
                    (struct sockaddr*)&sIotcBroadcast.server_addr, sizeof(sIotcBroadcast.server_addr)) < 0)
        {
            ERR_vPrintf(T_TRUE, "Send Data Error!\n");
        } else {
            BLUE_vPrintf(DBG_BROAD, "Send Data: %s\n", paResponse);
        }
        
        sleep(0);
    }
```

手机端代码，因为广播的数据会马上被路由转发，所以手机自己会收到自己的数据，需要重复收听直到收到服务端响应：

```text
public void run() {
            Message msgSocket = new Message();
            utils.DBG_vPrintf("Create Broadcast Socket");
            try {
                DatagramSocket uSocket = new DatagramSocket(null);
                uSocket.setReuseAddress(true);
                uSocket.bind(new InetSocketAddress(6789));


                byte[] buffer = new byte[40];
                DatagramPacket dataPacket = new DatagramPacket(buffer, buffer.length);
                byte[] data = stringSearch.getBytes();
                dataPacket.setData(data);
                dataPacket.setLength(data.length);
                dataPacket.setPort(6789);


                InetAddress broadcastAddr = InetAddress.getByName("255.255.255.255");
                dataPacket.setAddress(broadcastAddr);


                uSocket.send(dataPacket);
                uSocket.setSoTimeout(2000);//Set Recv Timeout


                byte[] byteRev = new byte[512];
                dataPacket = new DatagramPacket(byteRev,byteRev.length);
                utils.DBG_vPrintf("Recv Data From Server");
                uSocket.setSoTimeout(2000);


                while(true){
                    uSocket.receive(dataPacket);
                    String stringIotcResponse;
                    stringIotcResponse = new String(dataPacket.getData()).trim();
                    utils.DBG_vPrintf("Recv Data From Server Success:" + stringIotcResponse);
                    if (stringIotcResponse.equals("This is Server")){
                        break;
                    }
                }


                Message msgIotcAddress = new Message();
                msgIotcAddress.what = Utils.iFindServer;
                msgIotcAddress.obj = dataPacket.getAddress().getHostAddress();
                utils.DBG_vPrintf("The Server Ip is " + msgIotcAddress.obj.toString());


                handlerSocketRev.sendMessage(msgIotcAddress);


                uSocket.disconnect();
                uSocket.close();
            } catch (IOException e) {
                utils.ERR_vPrintf("Can't Search Iotc," + e.toString());
                e.printStackTrace();
                handlerSocketRev.sendEmptyMessage(Utils.iTimeOut);
            }


        }
```

