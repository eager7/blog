# 助记词到地址

\[TOC\]\#\# 助记词到地址![](file:///private/var/folders/yr/dk8nn_m51w901z1b3g22yhhw0000gn/T/WizNote/e0ecdf7d-4e2c-4265-ba2f-6f1a46d872e1/index_files/63455360.png)![](file:///private/var/folders/yr/dk8nn_m51w901z1b3g22yhhw0000gn/T/WizNote/e0ecdf7d-4e2c-4265-ba2f-6f1a46d872e1/index_files/63475514.png)

![](../.gitbook/assets/image%20%2816%29.png)

![](../.gitbook/assets/image%20%2814%29.png)

步骤为熵--&gt;熵+校验和--&gt;拆分--&gt;对照助记词本生成助记词--&gt;助记词+盐--&gt;2048次哈希生成512比特种子。

种子可以生成256位（32字节，64个16进制字符）的私钥，私钥可以导出公钥，然后再哈希\(RIPEMD160\)并进行base58编码生成地址。![](file:///private/var/folders/yr/dk8nn_m51w901z1b3g22yhhw0000gn/T/WizNote/e0ecdf7d-4e2c-4265-ba2f-6f1a46d872e1/index_files/64214118.png)![](file:///private/var/folders/yr/dk8nn_m51w901z1b3g22yhhw0000gn/T/WizNote/e0ecdf7d-4e2c-4265-ba2f-6f1a46d872e1/index_files/64243065.png)  
  


![](../.gitbook/assets/image%20%2815%29.png)



