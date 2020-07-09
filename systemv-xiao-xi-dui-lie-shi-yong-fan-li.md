# SystemV消息队列使用范例

在解读BLE的server程序时，发现它内部使用了System v的消息队列作为通信机制，因此在此研究一下此通信机制极其效率。 消息队列通常作为进程间通信的机制，即IPC（Inter-Progress Communication），IPC通常有四种形式：

* 1. 消息传递（管道，FIFO，消息队列）
* 1. 同步（互斥量，条件变量，读写锁，文件和记录锁，信号量）
* 1. 共享内存（匿名的和具名的）
* 1. 远程过程调用（Solaris门和Sun RPC）

上面这四种是最常见的非网络IPC，网络IPC通常使用TCP/IP协议进行开发。

System V使用消息队列标识符标识，即是一种基于消息传递的IPC。

主要函数：

```text
#include <sys/msg.h>
int msgget(key_t key, int oflag);
成功返回非负标识符，失败返回-1
```

返回值可以作为消息队列的ID，它是基于key产生的，key既可以是ftok的返回值，也可以是常值IPC\_PRIVATE。 oflag是读写权限的组合值，可以和IPC\_CREAT，IPC\_EXCL组合。规则如下：

* 设置oflag为IPC\_CREAT，如果IPC对象不存在，则创建一个新的IPC对象，否则返回该对象
* 设置oflag为IPC\_CREAT和IPC\_EXCL，如果对象不存在则创建一个新的对象，否则返回对象已存在的错误

单独设置oflag为IPC\_EXCL没有意义   

![](.gitbook/assets/image%20%2810%29.png)

![](.gitbook/assets/image%20%287%29.png)



oflag的读写权限规则如下： 

![](.gitbook/assets/image%20%285%29.png)

在使用msgget打开一个消息队列后，使用msgsend来发送消息：

```text
#include <sys/msg.h>
int msgsend(int msgid, const void *ptr, size_t length, int flag);
成功返回0，失败返回-1
```

msgid是消息队列ID，ptr是一个结构指针，具有如下模版：

```text
struct msgbuf{
    long mtype;
    char mtext[1];
}
```

消息类型必须大于0. flag可以是0或者IPC\_NOWAIT，后者可以实现msgsend的非阻塞调用，即在出错时立即返回一个ENGAIN错误，主要有下面两种情形：

* 1.指定的队列中有太多字节（队列msqid\_ds中msg\_abytes值）
* 2.系统范围内存在太多消息

如果flag为0，那么在上面两种情况下线程将被挂起。

消息接收

```text
#include <sys/msg.h>
ssize_t msgrecv(int msgid, const void *ptr, size_t length, long type, int flag);
成功返回字节数，失败返回-1
```

ptr同send的参数，是一个模版的结构体，length表示接收的最大长度，不包括ptr中的长整型字段 type指定希望读到的消息类型：

* 1. type为0，返回队列中第一个消息
* 1. type大于0，返回队列中类型值为type的第一个消息
* 1. type小于0，返回队列中最小于type绝对值的第一个消息

