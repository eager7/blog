# 误删Linux内核后修复系统

\[TOC\]

## 问题

在启动vmware时，提示gcc版本不对，发现是通过内核来进行版本匹配的，因此就卸掉了新装的4.9内核，导致开机无法启动。

## 解决办法

首先用U盘进入系统，挂载上系统盘，然后将U盘里的两个文件拷贝到系统盘的boot目录下：

> 所需要的文件在安装U盘的casper文件夹中，名字是vmlinuz和initrd.lz

接下来重新开机，然后按esc进入memory检测界面，不要进入memory，直接按c进入boot命令行，然后输入下面命令：

```text
ls -l #查看系统盘的UUID
set root=(hd0,msdos9) #msdos9是上面命令查找出来的盘的分区，根据实际情况不同，是ext4分区
linux /vmlinuz root=UUID=xxx ro locale=zh_CN quiet splash #xxx是上面命令查出的UUID
initrd /initrd.lz
boot
```

系统重启后可以进入系统了，这时候重新安装内核，不然重启后又无法进入了，先从网上下载一个内核，或者直接用命令行获取一个内核：

> wget [http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.4.30/linux-image-4.4.30-040430-generic\_4.4.30-040430.201611010007\_amd64.deb](http://kernel.ubuntu.com/~kernel-ppa/mainline/v4.4.30/linux-image-4.4.30-040430-generic_4.4.30-040430.201611010007_amd64.deb)

执行sudo dpkg -i xxx.deb进行安装即可。 如果用apt-get的方式安装内核，还是不行，找不到启动文件。

