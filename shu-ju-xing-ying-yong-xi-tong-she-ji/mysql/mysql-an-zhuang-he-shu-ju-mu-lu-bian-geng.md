# mysql安装和数据目录变更

\[TOC\]

## Centos

### 下载仓库源

下载文件 [https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm](https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm) 然后安装到机器上：

```text
sudo yum localinstall platform-and-version-specific-package-name.rpm
```

### 编辑使用版本号

如果使用5.7版本，则编辑文件`/etc/yum.repos.d/mysql-community.repo`使能5.7版本，同时关闭默认的8.0版本：

```text
[mysql57-community]


name=MySQL 5.7 Community Server



baseurl=http://repo.mysql.com/yum/mysql-5.7-community/el/6/$basearch/



enabled=1



gpgcheck=1



gpgkey=[file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql](http://file///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql)
```

查看使能的版本号：

```text
yum repolist enabled | grep mysql
```

### 安装mysql

运行下面命令即可安装：

```text
sudo yum install mysql-community-server
```

### 修改数据存储目录

查看配置文件`/etc/my.cnf`，存储目录为`data_dir`：

```text
datadir=/var/lib/mysql


socket=/var/lib/mysql/mysql.sock



log-error=/var/log/mysqld.log



pid-file=/var/run/mysqld/mysqld.pid
```

直接修改这个参数通常会报错，因此我们可以建立一个目录的软链接到这个路径：

```text
 ln -s /LVM/mysql /var/lib/mysql
```

LVM是大磁盘挂载的目录，这里需要注意一个点，就是LVM必须挂载到根目录下，如果挂载到一个用户目录下，会报权限错误，无法启动mysql。 最后一步是修改`/var/lib/mysql`的权限：

```text
chown -R mysql:mysql /var/lib/mysql
```

### 启动mysql并修改密码

```text
service mysqld start
```

启动后查找临时密码：

```text
➜ build git:(master) grep 'temporary password' /var/log/mysqld.log


2019-06-10T01:47:56.972117Z 5 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: b!q-&nTRm6/Q
```

用临时密码登录mysql，然后修改密码：

```text
➜  build git:(master) mysql -uroot -p'b!q-&nTRm6/Q'
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'blockABC!2018';
Query OK, 0 rows affected (0.10 sec)
```

### 创建新用户

一般我们不能使用root账户进行操作，因此我们需要创建一个新的用户来执行数据库操作：

```text
mysql> CREATE USER 'eth'@'%' IDENTIFIED BY 'blockABC!2018';
Query OK, 0 rows affected (0.01 sec)


mysql> GRANT ALL ON *.* TO 'eth'@'%';
Query OK, 0 rows affected (0.01 sec)
```

