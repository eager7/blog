# Linux内核开发示例

\[TOC\]

## 内核驱动

### 结构体

* inode结构体

  inode结构体是用来在内核中表示一个设备的，用户层看不到这个结构体，我们需要初始化它内部的一个结构体cdev：

  ```text
  struct cdev *pcdev_reversion;
  pcdev_reversion = cdev_alloc(); //inode 结构的一个结构体，在内核内部表示设备
  pcdev_reversion->owner = THIS_MODULE;
  pcdev_reversion->ops = &file_ops;
  cdev_add(pcdev_reversion, dev, number_reversion);
  ```

  如果我们没有用动态创建的方式来创建这个结构体，那么就需要使用cdev\_init来初始化它，这里cdev\_alloc在分配后就立即初始化了设备，所以我们就不用再次初始化了。

  注销此设备：

  ```text
  cdev_del(pcdev_reversion);
  ```

* file结构体

  文件结构体是内核暴露给用户层的，用户层通过这个文件结构体来访问内核里的cdev设备：

  ```text
  static struct file_operations file_ops = {
    .owner                 = THIS_MODULE,
    .open                 = open_reversion,
    .write                 = write_reversion,
    .read                 = read_reversion,
    .unlocked_ioctl     = ioctl_reversion
  };
  ```

* 设备号结构体dev\_t

  这个结构体是一个32位的整数，里面存放了设备的设备号，设备号是所有设备在内核中的标识，是唯一键，为了避免冲突，我们使用动态分配的方式来注册设备号：

  ```text
        if(major_reversion){
        dev = MKDEV(major_reversion, minor_reversion);
        ret = register_chrdev_region(dev, number_reversion, REVERSION_NAME);
    } else {
        ret = alloc_chrdev_region(&dev, minor_reversion, number_reversion, REVERSION_NAME);
        major_reversion = MAJOR(dev);
    }
    if(ret < 0){
        printk(KERN_WARNING "reversion:can't get major %d\n", major_reversion);
        return ret;
    }
  ```

  释放设备号：

  ```text
  unregister_chrdev_region(MKDEV(major_reversion, minor_reversion), number_reversion);//unregion device
  ```

### 文件操作

#### 读操作

```text
ssize_t read_reversion(struct file *filp, char __user *buf, size_t count, loff_t *ppos)
{
    if(copy_to_user(buf, "kernel", sizeof("kernel"))){
        printk(KERN_ERR "reversion:can't set date to user space\n");
        return EFAULT;
    }
    return 0;
}
```

#### 写操作

```text
ssize_t write_reversion(struct file *filp, const char __user *buf, size_t count, loff_t *ppos)
{
    char auBuf[128] = {0};
    if(copy_from_user(auBuf, buf, count)){
        printk(KERN_ERR "reversion:can't get date form user space\n");
        return EFAULT;
    }
    printk(KERN_DEBUG "reversion:get data form user space:%s\n", auBuf);
    return 0;
}
```

#### 控制操作

```text
long ioctl_reversion(struct file *filp, unsigned int cmd, unsigned long arg)
{
    return 0;
}
```

## 用户层代码

### 创建设备

我们在将驱动加载后，就可以在/proc/devices中看到设备的设备号了：

```text
pct@ubuntu:~$ cat /proc/devices | grep reversion
246 reversion
```

然后我们根据这个设备号创建设备：

```text
mknod [OPTION]... NAME TYPE [MAJOR MINOR]
```

首先获取设备号，用awk命令可获取到

```text
pct@ubuntu:~$ cat /proc/devices | awk '{if($2=="reversion")print $1}' 
246
```

我们动态分配得到的是设备的主设备号，次设备号是我们代码里写的0，然后创建设备：

```text
pct@ubuntu:~$ sudo mknod /dev/reversion c 246 0
[sudo] password for pct: 
pct@ubuntu:~$ ls /dev/reversion -l
crw-r--r-- 1 root root 246, 0 Jan  3 17:24 /dev/reversion
```

创建成功。 代码如下：

```text
#define DEV "/dev/reversion"


int dev_file = 0;


int dev_open()
{
    if(access(DEV,F_OK)){
        FILE *fp = popen("cat /proc/devices | awk '{if($2==\"reversion\")print $1}'", "r");
        char auMajor[5] = {0};
        if(0 == fread(auMajor, 1, sizeof(auMajor), fp))
        {
            printf("fread error:%s\n", strerror(errno));
            return -1;
        }
        int iMajor = atoi(auMajor);
        printf("the device major is:%d\n", iMajor);

        char auCommand[256] = {0};
        snprintf(auCommand, sizeof(auCommand), "mknod /dev/reversion c %d 0", iMajor);
        printf("command:%s\n", auCommand);
        system(auCommand);
    }
    dev_file = open(DEV,O_RDWR);
    if(dev_file == -1){
        printf("open error:%s\n", strerror(errno));
        return -1;
    }


    return 0;
}
```

### 读取设备

读取设备的操作就和读取文件一样：

```text
    char auRead[256] = {0};
    if(read(dev_file, auRead, sizeof(auRead)) == -1){
        printf("read error:%s\n", strerror(errno));
        return -1;
    }
    printf("read:%s\n", auRead);
```

返回值是我们在内核中写入的东西。

