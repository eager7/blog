# ubuntu搭建全局代理

\[TOC\]

## 安装shadows

```text
sudo add-apt-repository ppa:hzwhuang/ss-qt5
sudo apt-get update
sudo apt-get install shadowsocks-qt5
```

配置服务器和本地端口，配置成1080，然后启动软件。

## 安装privoxy

```text
pct@Chandler:~$ sudo apt-get install privoxy
```

## 配置privoxy

```text
sudo vim /etc/privoxy/config
```

找到1337行

```text
forward-socks5t / 127.0.0.1:1080 .
```

## 启动privoxy

//开启privoxy 服务就行

```text
sudo service privoxy start
```

// 设置http 和 https 全局代理

```text
export http_proxy='http://localhost:8118'



export https_proxy='https://localhost:8118'
```

## 测试

wget www.google.com

如果把返回200 ，并且把google的首页下载下来了，那就是成功了

