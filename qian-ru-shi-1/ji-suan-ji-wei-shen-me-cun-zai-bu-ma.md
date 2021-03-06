# 计算机为什么存在补码

## 原码，反码和补码 <a id="wiz-toc-0-2055385485"></a>

下面是大学的《微机原理》课程教材定义三种概念：

### 原码 <a id="wiz-toc-1-1643322007"></a>

规定正数的符号位为 0，负数的符号位为 1，其他位按照一般的方法来表示数的绝对值。用这样的表示方法得到的就是数的原码。

### 反码 <a id="wiz-toc-2-363332332"></a>

对于一个带符号的数来说，正数的反码与其原码相同，负数的反码为其原码除符号位以外的各位按位取反。

### 补码 <a id="wiz-toc-3-930872110"></a>

正数的补码与其原码相同，负数的补码为其反码在最低位加 1。

#### 运算规则 <a id="wiz-toc-4-60263434"></a>

类似于分配率：

## 微机原理 <a id="wiz-toc-5-1473410734"></a>

一般来说，我们理解上面的几条定义就可以用补码进行运算，也知道计算机是如何进行数值运算的。  
 但我们还需要往里追究一下，为什么计算机内部只存储数字的补码？

### 加法器 <a id="wiz-toc-6-2099514474"></a>

> 在电子学中，加法器（英语：adder）是一种用于执行加法运算的数字电路部件，是构成电子计算机核心微处理器中算术逻辑单元的基础。在这些电子系统中，加法器主要负责计算地址、索引等数据。除此之外，加法器也是其他一些硬件，例如二进制数的乘法器的重要组成部分。

加法器是由逻辑电路组成，逻辑电路则是由一些开合状态组成，计算机使用二进制作为计算度量也是因为二进制可以非常容易的被硬件模拟。  
 请注意，**计算机里面，只有加法器，没有减法器，因此所有的减法运算，都必须用加法进行。**  
 这就解释了我们为什么要存储所有数字的补码，在上面的第四条`运算规则`中，可以看到两个数的减法可以通过补码转换成两个数的加法，因此补码存在的意义就是为了解决减法运算问题。  
 ![](https://pic3.zhimg.com/80/v2-06929629cefa9c542c57c6b29be82a11_720w.jpg)

### 符号位的由来 <a id="wiz-toc-7-876711173"></a>

网络上关于补码最主要的争论是符号位的由来，像微机原理和一些教材，都认定最高的符号位是规定的，即最高位是`0`表示正数，最高位为`1`表示负数。  
 而一些人认为符号位不是规定的，而是由补码推演出来的，我们可以用一个例子看一下。  
 首先，我们认为没有符号位的定义，然后计算7-9：

注意，7-9在二进制中本是减不了的，因为7比9小，我们假设从再高位借1，那么就能得到`1111 1110`，类似于补码计算时溢出位1被忽略一样，这里再拿回来。  
 那么7-9最后得到的是`1111 1110`，按照我们上面的定义，`-2`的补码是什么呢？

可以看到，-2的表示和7-9借位结果是一样的，因此可以认为符号位其实是补码运算规则的附加产品。  
 不过这种争议的意义并不大，我们只需要理解补码是将减法转换为加法即可，依照这个思路的话，教材的方式确实更容易理解。

