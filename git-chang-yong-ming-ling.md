# Git常用命令

\[TOC\]

## 1. 配置使用默认密码

```text
cd ~



echo "https://eager7:pct1197639@github.com" > .git-credentials



git config --global credential.helper store
可以看到~/.gitconfig文件，会多了一项：



[credential]



    helper = store
```

## 2. 使用vim作为默认的编辑器

```text
git config --global core.editor vim
```

## 3. 基本使用命令

### 3.1 查看本地版本的状态以及差异：

```text
pct@ubuntu-x86:~/ubuntu-x86/IOTC$ git status



On branch master



Your branch is up-to-date with 'origin/master'.



Changes to be committed:



  (use "git reset HEAD <file>..." to unstage)



    new file: Makefile



    new file: main.c



    new file: utils.h
```

### 3.2 增加到本地库：

```text
pct@ubuntu-x86:~/ubuntu-x86/IOTC$ git add .
```

### 3.3 提交到本地：

```text
pct@ubuntu-x86:~/ubuntu-x86/IOTC$ git commit -a



[master 36cad9b] Add Makefile & main.c



 3 files changed, 153 insertions(+)



 create mode 100755 Makefile



 create mode 100644 main.c



 create mode 100755 utils.h
```

这样提交会弹出一个vim的文本编辑界面，写入你的注释，第一行简略描述，空一行，然后详细描述。

### 3.4 提交到远程仓库，即网络上：

```text
pct@ubuntu-x86:~/ubuntu-x86/IOTC$ git push -u origin master 



Counting objects: 6, done.



Delta compression using up to 2 threads.



Compressing objects: 100% (5/5), done.



Writing objects: 100% (5/5), 1.79 KiB | 0 bytes/s, done.



Total 5 (delta 0), reused 0 (delta 0)



To https://github.com/eager7/IOTC



   6c85c21..36cad9b master -> master



Branch master set up to track remote branch master from origin.
```

### 3.5 下载一个分支

```text
git clone https://github.com/eager7/socket_comm -b epoll
```

### 3.6 建立一个分支

```text
在本地新建一个分支： git branch Branch1



切换到你的新分支: git checkout Branch1



将新分支发布在github上： git push origin Branch1



在本地删除一个分支： git branch -d Branch1



在github远程端删除一个分支： git push origin :Branch1 (分支名前的冒号代表删除)



更新本地分支：git pull origin Branch1
```

### 3.7 create a new repository on the command line

```text
echo "# my_bashrc" >> README.md



git init



git add README.md



git commit -m "first commit"



git remote add origin https://github.com/eager7/my_bashrc.git



git push -u origin master
```

## 4. 设置git配置

设置git全局设置：

```text
git config --global user.name "your_name" 



git config --global user.email "your_email"
```

需要取消git的全局设置:

```text
git config --global --unset user.name 



git config --global --unset user.email
```

针对每个项目，单独设置用户名和邮箱，设置方法如下：

```text
git config user.name "your_name" 



git config user.email "your_email"
```

