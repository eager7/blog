# CRC校验算法

串口协议中使用了CRC校验，这个算法以前学过，现在都忘记了，今天再次整理一下。 代码段：

```text
static uint8 u8SL_CalculateCRC(uint16 u16Type, uint16 u16Length, uint8 *pu8Data)
{
    int n;
    uint8 u8CRC = 0;
    u8CRC ^= (u16Type >> 8) & 0xff;
    u8CRC ^= (u16Type >> 0) & 0xff;

    u8CRC ^= (u16Length >> 8) & 0xff;
    u8CRC ^= (u16Length >> 0) & 0xff;
    for(n = 0; n < u16Length; n++)
    {
        u8CRC ^= pu8Data[n];
    }
    return(u8CRC);
}
```

CRC校验定义：

> CRC为[校验和](https://zh.wikipedia.org/wiki/%E6%A0%A1%E9%AA%8C%E5%92%8C)的一种，是两个字节数据流采用二进制除法（没有进位，使用[XOR](https://zh.wikipedia.org/wiki/XOR)来代替减法）相除所得到的余数。其中被除数是需要计算校验和的信息数据流的二进制表示；除数是一个长度为的预定义（短）的二进制数，通常用多项式的系数来表示。在做除法之前，要在信息数据之后先加上个0.

从定义上看，上面的代码并不像是CRC校验的实现方式啊。。。 代码的实现方式非常简单，就是将要发送的字符串的每一位都进行异或运算，最后得出一个值。这个应该不是CRC校验，但是也能起到校验的作用，因为中间任何一个字符变化，都会导致异或运算得到的结果不一样，当然，也有可能错进错出，但是概率非常小了。

