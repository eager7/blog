# linux补丁的创建和应用

## 创建补丁文件

主要是使用diff命令来创建补丁，单个文件使用下面命令：

> pct@pct:~/My/Test/patch$ diff -uN Makefile Makefile-sta &gt; my-patch.in

如果是多个文件可以加上命令选项r来递归子目录的文件 创建的my-patch.in文件内容如下：

```text
--- Makefile    2015-02-07 20:02:41.939368198 +0800
+++ Makefile-sta    2015-02-07 20:01:53.663366479 +0800
@@ -1,225 +1,225 @@
-EXTRA_CFLAGS = -Idrivers/net/wireless/rt2860v2/include -Idrivers/net/wireless/rt2860v2/ate/include
+EXTRA_CFLAGS = -I$(src)/../rt2860v2/include -I$(src)/../rt2860v2/ate/include

-obj-$(CONFIG_RT2860V2_STA) += rt2860v2_sta.o
+obj-m += mt7620-sta.o
```

那么它的作用是什么呢？ 运行

> pct@pct:~/My/Test/patch$ patch -p0 &lt; my-patch.in

看看效果 发现Makefile里的内容全部变成Makefile-sta里的内容了。 也就是说补丁的作用是将---号的文件内容改成+++号文件的内容

第一行表示你需要修改的文件，我们把它改成我们的文件：

```text
--- mt7620_wifi2716_all_dpa_20130426.orig/rt2860v2_sta/Makefile
+++ mt7620_wifi2716_all_dpa_20130426/rt2860v2_sta/Makefile
```

这样就可以用这个补丁了

## patch文件的结构

### 补丁头

补丁头是分别由---/+++开头的两行，用来表示要打补丁的文件。---开头表示旧文件，+++开头表示新文件。 一个补丁文件中的多个补丁 一个补丁文件中可能包含以---/+++开头的很多节，每一节用来打一个补丁。所以在一个补丁文件中可以包含好多个补丁。

### 块

块是补丁中要修改的地方。它通常由一部分不用修改的东西开始和结束。他们只是用来表示要修改的位置。他们通常以@@开始，结束于另一个块的开始或者一个新的补丁头。

### 块的缩进

块会缩进一列，而这一列是用来表示这一行是要增加还是要删除的。

### 块的第一列

+号表示这一行是要加上的。 -号表示这一行是要删除的。 没有加号也没有减号表示这里只是引用的而不需要修改。

当然，打补丁是有技巧的，如果有很多个补丁，我们可以使用shell循环进行自动打补丁：

```text
cd backfire && for i in $$(ls $(PATCHES_COMMON_DIR)); do patch -p1 < $(PATCHES_COMMON_DIR)/$$i; done
```

