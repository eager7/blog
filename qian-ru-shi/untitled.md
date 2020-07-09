# 一种可靠串口协议

\[TOC\]

## 协议

kiwi mini里的ble通信协议没有校验和类型，如果出错没法检测，log打印也难以辨认，非常不利于调试，因此需要重新定义一个新的协议并完成它。 我们可以借鉴Zigbee的串口通信协议，那么我们先看看Zigbee的做法和代码。  

![](../.gitbook/assets/image%20%2812%29.png)

协议包括开始，结束，类型，长度，校验和数据。 协议将低于0x10的字符作为功能字符，0x01作为开始，0x03作为结束，其他预留。 那么如果类型，长度，校验和数据中出现了低于0x10的数据，就将它和0x10做异或运算^=，并在前面先发送0x02作为标志。

## server端程序

```text
static teSL_Status eSL_WriteMessage(uint16 u16Type, uint16 u16Length, uint8 *pu8Data)
{
    int n;
    uint8 u8CRC;
    u8CRC = u8SL_CalculateCRC(u16Type, u16Length, pu8Data);
    DBG_vPrintf(DBG_SERIALLINK_COMMS, "(%d, %d, %02x)\n", u16Type, u16Length, u8CRC);
    if (verbosity >= 10)
    {
        char acBuffer[4096];
        int iPosition = 0, i;       
        iPosition = sprintf(&acBuffer[iPosition], "Host->Node 0x%04X (Length % 4d)", u16Type, u16Length);
        for (i = 0; i < u16Length; i++)
        {
            iPosition += sprintf(&acBuffer[iPosition], " 0x%02X", pu8Data[i]);
        }
        INF_vPrintf(T_TRUE, "%s\n", acBuffer);
    }
    /* Send start character */
    if (iSL_TxByte(T_TRUE, SL_START_CHAR) < 0) return E_SL_ERROR;
    /* Send message type */
    if (iSL_TxByte(T_FALSE, (u16Type >> 8) & 0xff) < 0) return E_SL_ERROR;
    if (iSL_TxByte(T_FALSE, (u16Type >> 0) & 0xff) < 0) return E_SL_ERROR;
    /* Send message length */
    if (iSL_TxByte(T_FALSE, (u16Length >> 8) & 0xff) < 0) return E_SL_ERROR;
    if (iSL_TxByte(T_FALSE, (u16Length >> 0) & 0xff) < 0) return E_SL_ERROR;
    /* Send message checksum */
    if (iSL_TxByte(T_FALSE, u8CRC) < 0) return E_SL_ERROR;
    /* Send message payload */ 
    for(n = 0; n < u16Length; n++)
    {      
        if (iSL_TxByte(T_FALSE, pu8Data[n]) < 0) return E_SL_ERROR;
    }
    /* Send end character */
    if (iSL_TxByte(T_TRUE, SL_END_CHAR) < 0) return E_SL_ERROR;
    return E_SL_OK;
}
static int iSL_TxByte(bool_t bSpecialCharacter, uint8 u8Data)
{
    if(!bSpecialCharacter && (u8Data < 0x10))
    {
        u8Data ^= 0x10;
        if (eSerial_Write(SL_ESC_CHAR) != E_SERIAL_OK) return -1;
        //DBG_vPrintf(DBG_SERIALLINK_COMMS, " 0x%02x", SL_ESC_CHAR);
    }
    //DBG_vPrintf(DBG_SERIALLINK_COMMS, " 0x%02x", u8Data);
    return eSerial_Write(u8Data);
}
teSerial_Status eSerial_Write(const uint8 data)
{
    int err, attempts = 0;   
    DBG_vPrintf(DGB_SERIAL, "TX 0x%02x\n", data);   
    err = write(serial_fd,&data,1);
    if (err < 0)
    {
        if (errno == EAGAIN)/*try again*/
        {
            for (attempts = 0; attempts <= 5; attempts++)
            {
                usleep(1000);
                err = write(serial_fd,&data,1);
                if (err < 0)
                {
                    if ((errno == EAGAIN) && (attempts == 5))
                    {
                        ERR_vPrintf(T_TRUE, "Error writing to module after %d attempts(%s)", attempts, strerror(errno));
                        exit(-1);
                    }
                }
                else
                {
                    break;
                }
            }
        }
        else
        {
            ERR_vPrintf(T_TRUE, "Error writing to module(%s)", strerror(errno));
            exit(-1);
        }
    }
    return E_SERIAL_OK;
}
```

## 接收程序

```text
static teSL_Status eSL_ReadMessage(uint16 *pu16Type, uint16 *pu16Length, uint16 u16MaxLength, uint8 *pu8Message)
{
    static teSL_RxState eRxState = E_STATE_RX_WAIT_START;
    static uint8 u8CRC;
    uint8 u8Data;
    static uint16 u16Bytes;
    static bool_t bInEsc = T_FALSE;
    while(bSL_RxByte(&u8Data))
    {
        DBG_vPrintf(DBG_SERIALLINK_COMMS, "0x%02x\n", u8Data);
        switch(u8Data)
        {
        case SL_START_CHAR:
            u16Bytes = 0;
            bInEsc = T_FALSE;
            DBG_vPrintf(DBG_SERIALLINK_COMMS, "RX Start\n");
            eRxState = E_STATE_RX_WAIT_TYPEMSB;
            break;
        case SL_ESC_CHAR:
            DBG_vPrintf(DBG_SERIALLINK_COMMS, "Got ESC\n");
            bInEsc = T_TRUE;
            break;
        case SL_END_CHAR:
            DBG_vPrintf(DBG_SERIALLINK_COMMS, "Got END\n");

            if(*pu16Length > u16MaxLength)
            {
                /* Sanity check length before attempting to CRC the message */
                DBG_vPrintf(DBG_SERIALLINK_COMMS, "Length > MaxLength\n");
                eRxState = E_STATE_RX_WAIT_START;
                break;
            }

            if(u8CRC == u8SL_CalculateCRC(*pu16Type, *pu16Length, pu8Message))
            {
#if DBG_SERIALLINK
                int i;
                DBG_vPrintf(DBG_SERIALLINK, "RX Message type 0x%04x length %d: { ", *pu16Type, *pu16Length);
                for (i = 0; i < *pu16Length; i++)
                {
                    printf("0x%02x ", pu8Message[i]);
                }
                printf("}\n");
#endif /* DBG_SERIALLINK */

                eRxState = E_STATE_RX_WAIT_START;
                return E_SL_OK;
            }
            DBG_vPrintf(DBG_SERIALLINK_COMMS, "CRC BAD\n");
            break;
        default:
            if(bInEsc)
            {
                u8Data ^= 0x10;
                bInEsc = T_FALSE;
            }
            switch(eRxState)
            {
                case E_STATE_RX_WAIT_START:
                    break;                  
                case E_STATE_RX_WAIT_TYPEMSB:
                    *pu16Type = (uint16)u8Data << 8;
                    eRxState++;
                    break;
                case E_STATE_RX_WAIT_TYPELSB:
                    *pu16Type += (uint16)u8Data;
                    eRxState++;
                    break;
                case E_STATE_RX_WAIT_LENMSB:
                    *pu16Length = (uint16)u8Data << 8;
                    eRxState++;
                    break;
                case E_STATE_RX_WAIT_LENLSB:
                    *pu16Length += (uint16)u8Data;
                    DBG_vPrintf(DBG_SERIALLINK_COMMS, "Length %d\n", *pu16Length);
                    if(*pu16Length > u16MaxLength)
                    {
                        DBG_vPrintf(DBG_SERIALLINK_COMMS, "Length > MaxLength\n");
                        eRxState = E_STATE_RX_WAIT_START;
                    }
                    else
                    {
                        eRxState++;
                    }
                    break;
                case E_STATE_RX_WAIT_CRC:
                    DBG_vPrintf(DBG_SERIALLINK_COMMS, "CRC %02x\n", u8Data);
                    u8CRC = u8Data;
                    eRxState++;
                    break;
                case E_STATE_RX_WAIT_DATA:
                    if(u16Bytes < *pu16Length)
                    {
                        DBG_vPrintf(DBG_SERIALLINK_COMMS, "Data\n");
                        pu8Message[u16Bytes++] = u8Data;
                    }
                    break;
                default:
                    DBG_vPrintf(DBG_SERIALLINK_COMMS, "Unknown state\n");
                    eRxState = E_STATE_RX_WAIT_START;
            }
            break;
        }
    }
    return E_SL_NOMESSAGE;
}
static bool_t bSL_RxByte(uint8 *pu8Data)
{
    if (eSerial_Read(pu8Data) == E_SERIAL_OK)
    {
        return T_TRUE;
    }
    else
    {
        return T_FALSE;
    }
}
teSerial_Status eSerial_Read(uint8 *data)
{
    signed char res;
    res = read(serial_fd,data,1);
    if (res > 0)
    {
        DBG_vPrintf(DGB_SERIAL, "RX 0x%02x\n", *data);
    }
    else
    {
        //printf("Serial read: %d\n", res);
        if (res == 0)
        {
            //daemon_log(LOG_ERR, "Serial connection to module interrupted");
            //bRunning = 0;
        }
        return E_SERIAL_NODATA;
    }
    return E_SERIAL_OK;
}
```

## CRC校验部分

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

单片机端程序如下： log打印程序，我们使用一个类型作为log，把单片机端的打印输出到server端打印出来：

```text
void vSL_LogSend(uint8 u8LogLen, const char *pu8LogBuffer)
{
    uint8 n = 0;
    uint8 u8CRC = 0;
    uint8 u8Length = 0;
    uint8 u8LogStart = 0;
    // = strlen(pu8LogBuffer);
    uint8 u8Temp = 0;
    while ((u8LogLen - u8LogStart) != 0)
    {
        /* Send start character */
        vSL_TxByte(TRUE, SL_START_CHAR);
        /* Send message type */
        vSL_TxByte(FALSE, (E_SL_MSG_LOG >> 8) & 0xff);
        vSL_TxByte(FALSE, (E_SL_MSG_LOG >> 0) & 0xff);
        u8CRC = ((E_SL_MSG_LOG >> 8) & 0xff) ^ ((E_SL_MSG_LOG >> 0) & 0xff);
        for (u8Length = 0; u8Temp |= '\0'; u8Length++)
        {
            u8Temp = pu8LogBuffer[(u8LogStart + u8Length) & 0xFF];
            u8CRC ^= u8Temp;
        }
        u8CRC ^= 0;
        u8CRC ^= u8Length;
        /* Send message length */
        vSL_TxByte(FALSE, 0);
        vSL_TxByte(FALSE, u8Length);
        /* Send message checksum */
        vSL_TxByte(FALSE, u8CRC);
        /* Send message payload */
        for(n = 0; n < u8Length; n++)
        {
            vSL_TxByte(FALSE, pu8LogBuffer[u8LogStart]);
            u8LogStart++;
        }
        u8LogStart++;
        /* Send end character */
        vSL_TxByte(TRUE, SL_END_CHAR);
    }
}
int strlen(const char *str)
{
    //assert(str);
    if(*str==NULL)
    return 0;
    else
    return ( 1 + strlen( ++str ));
}
void vSL_TxByte(bool bSpecialCharacter, uint8 u8Data)
{
    if(!bSpecialCharacter && u8Data < 0x10)
    {
        /* Send escape character and escape byte */
        u8Data ^= 0x10;
        DebugWriteUint8(SL_ESC_CHAR);
    }
    DebugWriteUint8(u8Data);
}
```

## 串口的接收解码：

```text
static int eSL_ReadMessage(uint16 *pu16Type, uint16 *pu16Length, uint16 u16MaxLength, uint8 *pu8Message, uint8 *psRecv)
{
    static teSL_RxState eRxState = E_STATE_RX_WAIT_START;
    static uint8 u8CRC;
    uint8 u8Data;
    static uint16 u16Bytes;
    static bool bInEsc = FALSE;
    while(*psRecv)
    {
        u8Data = *psRecv;
        psRecv++;
        switch(u8Data)
        {
        case SL_START_CHAR:
            u16Bytes = 0;
            bInEsc = FALSE;
            eRxState = E_STATE_RX_WAIT_TYPEMSB;
            break;
        case SL_ESC_CHAR:
            bInEsc = TRUE;
            break;
        case SL_END_CHAR:
            if(*pu16Length > u16MaxLength)
            {
                /* Sanity check length before attempting to CRC the message */
                eRxState = E_STATE_RX_WAIT_START;
                break;
            }
            if(u8CRC == u8SL_CalculateCRC(*pu16Type, *pu16Length, pu8Message))
            {
                eRxState = E_STATE_RX_WAIT_START;
                return 0;
            }
            break;
        default:
            if(bInEsc)
            {
                u8Data ^= 0x10;
                bInEsc = FALSE;
            }
            switch(eRxState)
            {
                case E_STATE_RX_WAIT_START:
                    break;
                case E_STATE_RX_WAIT_TYPEMSB:
                    *pu16Type = (uint16)u8Data << 8;
                    eRxState++;
                    break;
                case E_STATE_RX_WAIT_TYPELSB:
                    *pu16Type += (uint16)u8Data;
                    eRxState++;
                    break;
                case E_STATE_RX_WAIT_LENMSB:
                    *pu16Length = (uint16)u8Data << 8;
                    eRxState++;
                    break;
                case E_STATE_RX_WAIT_LENLSB:
                    *pu16Length += (uint16)u8Data;
                    if(*pu16Length > u16MaxLength)
                    {
                        eRxState = E_STATE_RX_WAIT_START;
                    }
                    else
                    {
                        eRxState++;
                    }
                    break;
                case E_STATE_RX_WAIT_CRC:
                    u8CRC = u8Data;
                    eRxState++;
                    break;
                case E_STATE_RX_WAIT_DATA:
                    if(u16Bytes < *pu16Length)
                    {
                        pu8Message[u16Bytes++] = u8Data;
                    }
                    break;
                default:
                    eRxState = E_STATE_RX_WAIT_START;
            }
            break;
        }
    }
    return 1;
}
```

## 串口写数据：

```text
void vSL_WriteMessage(uint16 u16Type, uint16 u16Length, uint8 *pu8Data)
{
    uint16 n;
    uint8 u8CRC;
    u8CRC = u8SL_CalculateCRC(u16Type, u16Length, pu8Data);
    /* Send start character */
    vSL_TxByte(TRUE, SL_START_CHAR);
    /* Send message type */
    vSL_TxByte(FALSE, (u16Type >> 8) & 0xff);
    vSL_TxByte(FALSE, (u16Type >> 0) & 0xff);
    /* Send message length */
    vSL_TxByte(FALSE, (u16Length >> 8) & 0xff);
    vSL_TxByte(FALSE, (u16Length >> 0) & 0xff);
    /* Send message checksum */
    vSL_TxByte(FALSE, u8CRC);
    /* Send message payload */
    for(n = 0; n < u16Length; n++)
    {
        vSL_TxByte(FALSE, pu8Data[n]);
    }
    /* Send end character */
    vSL_TxByte(TRUE, SL_END_CHAR);
}
```

CRC校验使用同样程序。

