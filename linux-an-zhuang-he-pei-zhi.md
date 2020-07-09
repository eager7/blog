# linux安装和配置

\[TOC\]

## 1. 修改root密码

```text
sudo passwd root
```

## 2. 安装ssh

```text
apt-get install openssh-server
```

## 3. 连接securecrt

```text
用户名：root 密码：pct
```

## 4. 安装图形界面

```text
apt-get install x-window-system-core 
apt-get install gnome-core
startx
```

## 5. 配置samba

```text
apt-get install samba
设置虚拟机连接方式为桥接
尝试Windows和Linux之间能否互相ping通
修改samba文件-- vi /etc/samba/smb.conf
在====Share Definitions====下添加
```

```text
[pct]
        comment = Root/Directories
        browseable = yes
        writable = yes
        path = /
        valid users = pct
```

```text
[pct]
        path = /home/pct
        writeable = yes
        valid users = pct
        commet = Home Directories
```

```text
    修改smb密码: sudo smbpasswd -a pct
重启smb: restart smbd
连接//10.128.118.50

当连接不上时，关闭Linux防火墙service iptables stop
```

## 6. 安装远程桌面

```text
apt-get install xrdp
然后启动附件里的连接远程桌面就可以了
```

## 7. 添加一个新用户

```text
首先添加组，
sudo groupadd user
然后添加用户
sudo useradd -g user -s /bin/bash -m pct 
指定用户目录的添加方式：sudo useradd -g user -s /bin/bash -m liutao -d /home/extend/liutao
再改密码
sudo passwd pct
最后将此用户添加到samba中
sudo smbpasswd -a pct
```

## 8. 将user组加入root权限

```text
vi /etc/sudoers
找到这行 root ALL=(ALL) ALL,在他下面添加xxx ALL=(ALL) ALL (这里的xxx是你的用户名)
ps:这里说下你可以sudoers添加下面四行中任意一条
youuser            ALL=(ALL)                ALL
%youuser           ALL=(ALL)                ALL
youuser            ALL=(ALL)                NOPASSWD: ALL
%youuser           ALL=(ALL)                NOPASSWD: ALL
第一行:允许用户youuser执行sudo命令(需要输入密码).
第二行:允许用户组youuser里面的用户执行sudo命令(需要输入密码).
第三行:允许用户youuser执行sudo命令,并且在执行的时候不输入密码.
第四行:允许用户组youuser里面的用户执行sudo命令,并且在执行的时候不输入密码.    
```

## 9. 更改根目录下目录的用户和组

```text
使用chown命令
chown duanw:user -R /SVN
将SVN目录改为duanw的私人目录
```

## 10. 建立共享目录

```text
Jenkins 目录是编译的目录，用户组所有人都可以访问，切换到duanw用户下，为这个文件夹添加组的写权限即可
chmod g+w -R /Jenkins
```

## 11. 添加samba共享目录

```text
在samba的配置文件/etc/samba/smb.conf下添加下面配置：
```

```text
```C,default
[share]<div><br/></div>        comment = share all<div><br/></div>        path = /home/Share<div><br/></div>        browseable = yes<div><br/></div>        public = yes<div><br/></div>        writable = no
```

```text
然后在/home下建立Share目录，然后更改其权限：chmod 777 Share
重启samba服务：sudo /etc/init.d/smbd restart


##### 12. 配置tftp服务
首先需要配置TFTP服务器，
配置Ubuntu tftp服务的步骤：
###### 1、安装相关软件包：Ubuntu tftp（服务端），tftp（客户端），xinetd
```C,default
sudo apt-get install tftpd tftp xinetd
```

### 2、建立配置文件

在/etc/xinetd.d/下建立一个配置文件tftp sudo vi tftp 在文件中输入以下内容：

```text
```C,default
service tftp<div><br/></div>{socket_type = dgram<div><br/></div>protocol = udp<div><br/></div>wait = yes<div><br/></div>user = root<div><br/></div>server = /usr/sbin/in.tftpd<div><br/></div>server_args = -s /tftpboot<div><br/></div>disable = no<div><br/></div>per_source = 11<div><br/></div>cps = 100 2<div><br/></div>flags = IPv4}
```

```text
###### 3、建立Ubuntu tftp服务文件目录（上传文件与下载文件的位置），并且更改其权限
```C,default
sudo mkdir /tftpboot<div><br/></div>sudo chmod 777 /tftpboot -R
```

### 4、重新启动服务

```text
sudo /etc/init.d/xinetd restart
```

自测： 在tftpboot目录下创建一个文件，然后在其他目录下将这个文件取回：

```text
pct@ubuntu-x64:~$ tftp 127.0.0.1
tftp> get 1
tftp> quit
```

## 13. 挂载windows共享文件夹

有时候我们需要将Windows的共享文件夹挂载到Linux系统上，进行文件拷贝或者同步 首先需要将Windows的文件夹设为共享文件夹  然后在Linux端挂载：

```text
sudo mount -t cifs -o file_mode=0777,dir_mode=0777,username=panch_000,password=pct1197639 //10.128.118.22/Work /home/pct/win10/
```

需要注意的是，Work目录并不是绝对路径，而是你分享的那个文件夹的名字，不必理会它的路径。 而且用户名并不是登录时显示的那个用户名，这个可能是Windows系统给你默认分配的一个，例如我的信息本来是这样的：  但是实际上我的用户名是这样的：  panch\_000才是我真正的用户名，这个需要注意。

## 14. 挂载Linux samba共享目录

先安装cifs支持：

```text
sudo apt install cifs-utils
```

然后挂载：

```text
sudo mount.cifs -o username=firefly,password=firefly //192.168.4.10/firefly firefly/
```

## 15. 安装搜狗用gdebi

## 16. 安装Courier New字体

> apt-get install ttf-mscorefonts-installer

安装的时候会出现一个协议 按TAB键 ，可以选中&lt;确定&gt;按钮（有些会看不到确定按钮），按Enter 。

## 17. 双系统启动选项设置

1. 进入Ubuntu，打开/etc/default/grub文件 sudu gedit /etc/default/grub
2. 修改GRUB\_DEFAULT = X（默认为0）

   > X的值可以这样计算：打开/boot/grub/grub.cfg文件，其中包含了开机菜单中所有启动项的名称，格式如：menuentry 'Ubuntu, whith Linux 2.6.35-25-generic'，所有启动项名称以menuentry打头。找到windows启动项的序号，这个序号减1的值即为X的值。

3. 最后一步，sudo update-grub，更新/boot/grub/grub.cfg文件

## 17.安装显卡驱动

在intel官网下载驱动安装工具，然后安装上运行

> [https://01.org/zh/linuxgraphics/downloads?langredirect=1](https://01.org/zh/linuxgraphics/downloads?langredirect=1)

```text
sudo apt-get install fonts-ancient-scripts
sudo apt-get install ttf-ancient-fonts
sudo dpkg -i intel-graphics-update-tool_2.0.2_amd64.deb
```

## 18. Install synergy

```text
git clone https://github.com/symless/synergy
./hm.sh conf -g 1
./hm.sh build
sudo apt-get install fakeroot
./hm.sh package deb
```

