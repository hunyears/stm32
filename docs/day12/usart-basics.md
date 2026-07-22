# Day 12 - USART 串口基础学习笔记

> 学习日期：2026-07-28

---

## 一、核心知识点

### 1.1 什么是串口？

#### 定义

串口（USART，Universal Synchronous/Asynchronous Receiver Transmitter）是一种常用的通信接口，用于设备间的数据传输。

```
串口通信示意：

   STM32                        电脑
┌─────────┐                  ┌─────────┐
│         │    TX ────────▶  │   RX    │
│  USART  │                  │  USB转  │
│         │    RX ◀────────  │   TTL   │
│         │                  │         │
│    GND  │─────┴──────────  │   GND   │
└─────────┘                  └─────────┘

核心要点：
    TX（发送）连接对方的 RX（接收）
    RX（接收）连接对方的 TX（发送）
    GND 必须共地！
```

#### 串口 vs 并口

| 特性 | 串口 | 并口 |
|------|------|------|
| 数据线 | 1-2 根 | 8-32 根 |
| 速度 | 较慢 | 较快 |
| 距离 | 远 | 近 |
| 成本 | 低 | 高 |
| 应用 | 调试、通信 | 内部总线 |

### 1.2 串口通信参数

```
┌─────────────────────────────────────────────────────┐
│  串口通信关键参数                                   │
├─────────────────────────────────────────────────────┤
│  波特率   │  数据传输速率（如 9600、115200 bps）   │
│  数据位   │  每帧数据的位数（通常 8 位）           │
│  停止位   │  帧结束标志（1 或 2 位）               │
│  校验位   │  奇偶校验（可选）                       │
│  流控    │  硬件流控/软件流控（可选）               │
└─────────────────────────────────────────────────────┘

常用配置：115200-8-N-1
    - 波特率：115200 bps
    - 数据位：8 位
    - 校验位：无（N = None）
    - 停止位：1 位
```

### 1.3 串口数据帧格式

```
一个字节数据的传输格式（8-N-1）：

    起始位  数据位（低位在前）      停止位
      ↓    D0 D1 D2 D3 D4 D5 D6 D7   ↓
      ┌   ┌─┐┌─┐┌─┐┌─┐┌─┐┌─┐┌─┐┌─┐ ┌
空闲  │   │ ││ ││ ││ ││ ││ ││ ││ │ │ │
──────┘   └─┘└─┘└─┘└─┘└─┘└─┘└─┘└─┘ └──────

时间轴 →

一位数据时间 = 1 / 波特率
示例（115200 bps）：一位时间 ≈ 8.68 μs
传输一个字节 ≈ 86.8 μs（含起始位和停止位）
```

### 1.4 STM32 的 USART

```
STM32F103 串口资源：

USART1：TX(PA9)、RX(PA10)  ← APB2 总线
USART2：TX(PA2)、RX(PA3)   ← APB1 总线
USART3：TX(PB10)、RX(PB11) ← APB1 总线

注意：
    - USART1 在 APB2 总线，速度更快
    - 其他 USART 在 APB1 总线
```

---

## 二、常用库函数

### 2.1 USART 初始化

```c
// 开启 USART 时钟
RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);  // USART1
RCC_APB1PeriphClockCmd(RCC_APB1Periph_USART2, ENABLE);  // USART2

// USART 初始化
void USART_Init(USART_TypeDef* USARTx, USART_InitTypeDef* USART_InitStruct);

// 结构体定义
typedef struct {
    uint32_t USART_BaudRate;    // 波特率
    uint16_t USART_WordLength;  // 数据位长度
    uint16_t USART_StopBits;    // 停止位
    uint16_t USART_Parity;      // 校验位
    uint16_t USART_Mode;        // 模式（TX/RX）
    uint16_t USART_HardwareFlowControl;  // 硬件流控
} USART_InitTypeDef;

// 使能 USART
void USART_Cmd(USART_TypeDef* USARTx, FunctionalState NewState);
```

### 2.2 数据发送

```c
// 发送单个字节
void USART_SendData(USART_TypeDef* USARTx, uint16_t Data);

// 检查发送完成标志
FlagStatus USART_GetFlagStatus(USART_TypeDef* USARTx, uint16_t USART_FLAG);

// 常用标志
USART_FLAG_TXE   // 发送数据寄存器空
USART_FLAG_TC    // 发送完成
```

### 2.3 数据接收

```c
// 接收单个字节
uint16_t USART_ReceiveData(USART_TypeDef* USARTx);

// 检查接收标志
USART_FLAG_RXNE  // 接收数据寄存器非空
```

### 2.4 中断相关

```c
// 使能中断
void USART_ITConfig(USART_TypeDef* USARTx, uint16_t USART_IT, FunctionalState NewState);

// 常用中断
USART_IT_RXNE   // 接收中断
USART_IT_TXE    // 发送中断
USART_IT_TC     // 发送完成中断
USART_IT_IDLE   // 空闲中断

// 获取中断状态
ITStatus USART_GetITStatus(USART_TypeDef* USARTx, uint16_t USART_IT);

// 清除中断标志
void USART_ClearITPendingBit(USART_TypeDef* USARTx, uint16_t USART_IT);
```

---

## 三、核心原理

### 3.1 波特率计算

```
波特率 = Fclk / (16 × USARTDIV)

其中：
    Fclk = USART 时钟频率
    USARTDIV = BRR 寄存器值（16位）

示例（USART1，72MHz，115200 bps）：
    USARTDIV = 72000000 / (16 × 115200) = 39.0625
    
    BRR 整数部分 = 39 = 0x27
    BRR 小数部分 = 0.0625 × 16 = 1 = 0x01
    
    BRR = 0x271

标准库会自动计算，只需设置：
    USART_InitStructure.USART_BaudRate = 115200;
```

### 3.2 发送流程

```
发送一个字节：

步骤1：等待 TXE 标志（发送寄存器空）
    while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);

步骤2：写入数据到 DR 寄存器
    USART_SendData(USART1, data);

步骤3：等待 TC 标志（发送完成）
    while (USART_GetFlagStatus(USART1, USART_FLAG_TC) == RESET);

DR（数据寄存器）结构：
    ┌─────────────────────────────┐
    │  TDR（发送）   │  RDR（接收）│  ← 实际是同一个地址
    └─────────────────────────────┘
    
    写入时 → 发送到 TDR
    读取时 → 从 RDR 读取
```

### 3.3 接收流程

```
接收一个字节：

轮询方式：
    while (USART_GetFlagStatus(USART1, USART_FLAG_RXNE) == RESET);
    data = USART_ReceiveData(USART1);

中断方式：
    1. 配置接收中断
    2. 数据到达时触发中断
    3. 在中断中读取数据
```

---

## 四、电路连接

### 4.1 硬件清单

| 组件 | 数量 | 说明 |
|------|------|------|
| STM32F103C8T6 最小系统板 | 1 | - |
| USB 转 TTL 模块（CH340） | 1 | 串口通信 |
| 杜邦线 | 若干 | 连接电路 |

### 4.2 推荐接线方案

| STM32 引脚 | USB-TTL 模块 | 说明 |
|------------|--------------|------|
| PA9 (TX) | RX | 发送 → 接收 |
| PA10 (RX) | TX | 接收 ← 发送 |
| GND | GND | 共地 |

### 4.3 电路图

```
    STM32F103C8T6              USB-TTL(CH340)
   ┌─────────────┐            ┌─────────────┐
   │             │            │             │
   │    PA9(TX) ─┼────────────┼─ RXD        │
   │             │            │             │
   │   PA10(RX) ─┼────────────┼─ TXD        │
   │             │            │             │
   │        GND ─┼────────────┼─ GND        │
   │             │            │             │
   └─────────────┘            │     USB ────┼─▶ 电脑
                              └─────────────┘

重要：交叉连接！
    STM32 的 TX → USB-TTL 的 RX
    STM32 的 RX → USB-TTL 的 TX
```

---

## 五、完整代码实现

### 5.1 实验一：串口发送（printf 重定向）

```c
#include "stm32f10x.h"
#include <stdio.h>

// ==================== 重定向 printf ====================
int fputc(int ch, FILE *f)
{
    USART_SendData(USART1, (uint8_t)ch);
    while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
    return ch;
}

// ==================== USART1 初始化 ====================
void USART1_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    USART_InitTypeDef USART_InitStructure;

    // 开启时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_USART1, ENABLE);

    // 配置 TX(PA9) 为复用推挽输出
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // 配置 RX(PA10) 为浮空输入
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // USART 配置
    USART_InitStructure.USART_BaudRate = 115200;
    USART_InitStructure.USART_WordLength = USART_WordLength_8b;
    USART_InitStructure.USART_StopBits = USART_StopBits_1;
    USART_InitStructure.USART_Parity = USART_Parity_No;
    USART_InitStructure.USART_Mode = USART_Mode_Tx | USART_Mode_Rx;
    USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    USART_Init(USART1, &USART_InitStructure);

    // 使能 USART
    USART_Cmd(USART1, ENABLE);
}

// ==================== 主函数 ====================
int main(void)
{
    USART1_Init();

    printf("Hello STM32!\r\n");
    printf("System started.\r\n");

    uint32_t count = 0;
    while (1)
    {
        printf("Count: %d\r\n", count++);
        for (uint32_t i = 0; i < 8000000; i++);  // 简单延时
    }
}
```

### 5.2 实验二：串口接收（轮询方式）

```c
#include "stm32f10x.h"
#include <stdio.h>

void USART1_Init(void)
{
    // ... 同上 ...
}

// 接收一个字节
uint8_t USART_ReceiveByte(void)
{
    while (USART_GetFlagStatus(USART1, USART_FLAG_RXNE) == RESET);
    return (uint8_t)USART_ReceiveData(USART1);
}

// 发送一个字节
void USART_SendByte(uint8_t data)
{
    USART_SendData(USART1, data);
    while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
}

// 发送字符串
void USART_SendString(const char *str)
{
    while (*str)
    {
        USART_SendByte(*str++);
    }
}

int main(void)
{
    uint8_t received;

    USART1_Init();
    printf("Ready to receive...\r\n");

    while (1)
    {
        received = USART_ReceiveByte();

        // 回显收到的字符
        printf("Received: %c (0x%02X)\r\n", received, received);

        // 特殊命令处理
        if (received == '1')
        {
            printf("LED ON\r\n");
        }
        else if (received == '0')
        {
            printf("LED OFF\r\n");
        }
    }
}
```

### 5.3 实验三：串口中断接收

```c
#include "stm32f10x.h"
#include <stdio.h>

#define RX_BUF_SIZE 64
uint8_t rx_buffer[RX_BUF_SIZE];
volatile uint8_t rx_index = 0;
volatile uint8_t rx_complete = 0;

void USART1_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    USART_InitTypeDef USART_InitStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

    // 开启时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_USART1, ENABLE);

    // GPIO 配置
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // USART 配置
    USART_InitStructure.USART_BaudRate = 115200;
    USART_InitStructure.USART_WordLength = USART_WordLength_8b;
    USART_InitStructure.USART_StopBits = USART_StopBits_1;
    USART_InitStructure.USART_Parity = USART_Parity_No;
    USART_InitStructure.USART_Mode = USART_Mode_Tx | USART_Mode_Rx;
    USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    USART_Init(USART1, &USART_InitStructure);

    // 使能接收中断
    USART_ITConfig(USART1, USART_IT_RXNE, ENABLE);

    // NVIC 配置
    NVIC_InitStructure.NVIC_IRQChannel = USART1_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    USART_Cmd(USART1, ENABLE);
}

// 中断服务函数
void USART1_IRQHandler(void)
{
    uint8_t data;

    if (USART_GetITStatus(USART1, USART_IT_RXNE) != RESET)
    {
        data = (uint8_t)USART_ReceiveData(USART1);

        if (data == '\r' || data == '\n')  // 收到换行符
        {
            rx_buffer[rx_index] = '\0';  // 字符串结束符
            rx_complete = 1;
        }
        else if (rx_index < RX_BUF_SIZE - 1)
        {
            rx_buffer[rx_index++] = data;
        }

        USART_ClearITPendingBit(USART1, USART_IT_RXNE);
    }
}

int main(void)
{
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    USART1_Init();

    printf("USART Interrupt Demo\r\n");
    printf("Type something and press Enter...\r\n");

    while (1)
    {
        if (rx_complete)
        {
            rx_complete = 0;

            printf("You typed: %s\r\n", rx_buffer);
            printf("Length: %d\r\n", rx_index);

            // 清空缓冲区
            rx_index = 0;
        }
    }
}
```

---

## 六、调试技巧

### 6.1 常见问题排查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| 乱码 | 波特率不匹配 | 检查两端波特率设置 |
| 无数据 | TX/RX 接反 | 交叉连接 TX-RX |
| 发送失败 | 时钟未开启 | 检查 RCC 配置 |
| 接收不到 | 中断未使能 | 检查 NVIC 配置 |

### 6.2 串口调试工具

```
常用工具：
    - 串口助手（Windows）
    - Tera Term
    - PuTTY
    - SecureCRT
    - minicom（Linux）

使用步骤：
    1. 连接 USB-TTL 到电脑
    2. 查看设备管理器，确认 COM 口号
    3. 打开串口调试软件
    4. 设置波特率等参数
    5. 打开串口，开始通信
```

---

## 七、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【串口连接】TX 接 RX，RX 接 TX，GND 共地           │
│  【常用配置】115200-8-N-1                           │
│  【发送流程】等待 TXE → 写数据 → 等待 TC           │
│  【接收流程】等待 RXNE → 读数据                     │
│  【中断方式】RXNE 中断触发接收                      │
└─────────────────────────────────────────────────────┘
```

### 记忆口诀

```
串口通信要交叉，TX 接 RX 别搞反。
波特率要两边对，115200 是标准。
发送等待 TXE 标志，接收等待 RXNE。
中断方式更高效，主程序不阻塞。
```

---

## 八、拓展练习

1. **串口命令解析**：实现简单的命令解析器（如 "LED ON"、"LED OFF"）
2. **串口控制 LED**：通过串口发送命令控制 LED
3. **数据打包传输**：实现带校验的数据包协议
4. **多机通信**：两个 STM32 通过串口通信

---

*上一篇：Day 11 - 输入捕获*  
*下一篇：Day 13 - USART 中断与 DMA*