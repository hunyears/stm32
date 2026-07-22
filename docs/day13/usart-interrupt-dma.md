# Day 13 - USART 中断与 DMA 学习笔记

> 学习日期：2026-07-29

---

## 一、核心知识点

### 1.1 为什么需要 DMA？

#### 串口传输的瓶颈

```
传统方式（CPU 搬运数据）：

发送 100 字节：
    CPU 执行 100 次 "写入 DR + 等待 TXE"
    CPU 被完全占用，无法做其他事

DMA 方式（直接内存访问）：

发送 100 字节：
    CPU 设置 DMA 源地址、目标地址、长度
    DMA 自动完成数据搬运
    CPU 可以做其他事
    完成后 DMA 通知 CPU

比喻：
    CPU = 快递员
    数据 = 快递包裹
    串口 = 快递站
    
    传统方式：快递员亲自送每个包裹
    DMA方式：快递员把包裹交给快递站，快递站自己送
```

### 1.2 DMA 简介

#### 什么是 DMA？

DMA（Direct Memory Access，直接内存访问）是一种**无需 CPU 参与**的数据传输方式。

```
DMA 工作原理：

┌──────────────────────────────────────────────────────┐
│                                                      │
│   内存（源）─────┐                      ┌───── 外设   │
│                 │     ┌──────────┐     │            │
│                 └────▶│   DMA    │─────┘            │
│                       │  控制器   │                  │
│   内存（目标）───┐     └──────────┘     ┌───── 外设   │
│                 │          │           │            │
│                 └──────────┴───────────┘            │
│                           │                         │
│                           ↓                         │
│                    DMA 完成后通知 CPU                │
│                                                      │
└──────────────────────────────────────────────────────┘
```

### 1.3 STM32 的 DMA

```
STM32F103 DMA 资源：

DMA1（7 个通道）：
    通道1：ADC1、TIM2_CH3、SPI1_RX
    通道2：SPI1_TX、USART3_TX
    通道3：USART3_RX
    通道4：USART1_TX、I2C1_TX
    通道5：USART1_RX、I2C1_RX
    通道6：USART2_TX
    通道7：USART2_RX

DMA2（5 个通道，仅大容量产品）：
    通道1~5：支持更多外设

USART1 的 DMA 映射：
    USART1_TX → DMA1 通道4
    USART1_RX → DMA1 通道5
```

---

## 二、常用库函数

### 2.1 DMA 初始化

```c
// 开启 DMA 时钟
RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);

// DMA 初始化
void DMA_Init(DMA_Channel_TypeDef* DMAy_Channelx, DMA_InitTypeDef* DMA_InitStruct);

// 结构体定义
typedef struct {
    uint32_t DMA_PeripheralBaseAddr;    // 外设地址
    uint32_t DMA_MemoryBaseAddr;        // 内存地址
    uint32_t DMA_DIR;                   // 传输方向
    uint32_t DMA_BufferSize;            // 数据长度
    uint32_t DMA_PeripheralInc;         // 外设地址递增
    uint32_t DMA_MemoryInc;             // 内存地址递增
    uint32_t DMA_PeripheralDataSize;    // 外设数据宽度
    uint32_t DMA_MemoryDataSize;        // 内存数据宽度
    uint32_t DMA_Mode;                  // 模式（普通/循环）
    uint32_t DMA_Priority;              // 优先级
    uint32_t DMA_M2M;                   // 内存到内存
} DMA_InitTypeDef;

// 使能 DMA 通道
void DMA_Cmd(DMA_Channel_TypeDef* DMAy_Channelx, FunctionalState NewState);

// 使能 DMA 中断
void DMA_ITConfig(DMA_Channel_TypeDef* DMAy_Channelx, uint32_t DMA_IT, FunctionalState NewState);
```

### 2.2 DMA 与 USART 配合

```c
// 使能 USART 的 DMA 发送/接收
void USART_DMACmd(USART_TypeDef* USARTx, uint16_t USART_DMAReq, FunctionalState NewState);

// DMA 请求
USART_DMAReq_Tx   // 发送 DMA 请求
USART_DMAReq_Rx   // 接收 DMA 请求
```

---

## 三、核心原理

### 3.1 DMA 传输方向

```
三种传输方向：

1. 外设 → 内存（如串口接收）
    DMA_DIR_PeripheralSRC
    外设地址 = &USART1->DR
    内存地址 = rx_buffer

2. 内存 → 外设（如串口发送）
    DMA_DIR_PeripheralDST
    内存地址 = tx_buffer
    外设地址 = &USART1->DR

3. 内存 → 内存（数据搬运）
    DMA_M2M_Enable
```

### 3.2 DMA 模式

```
普通模式（Normal）：
    传输完成后停止
    需要重新配置才能再次传输

循环模式（Circular）：
    传输完成后自动重新开始
    适合连续数据采集

DMA_Mode_Normal    // 普通模式
DMA_Mode_Circular  // 循环模式
```

### 3.3 DMA 中断

```
DMA 传输完成中断：
    DMA_IT_TC（Transfer Complete）
    所有数据传输完成

DMA 半传输中断：
    DMA_IT_HT（Half Transfer）
    一半数据传输完成

DMA 错误中断：
    DMA_IT_TE（Transfer Error）
    传输出错
```

---

## 四、电路连接

与 Day 12 相同，使用 USB-TTL 模块连接 USART1。

---

## 五、完整代码实现

### 5.1 实验一：DMA 发送

```c
#include "stm32f10x.h"
#include <stdio.h>

uint8_t tx_buffer[] = "Hello DMA!\r\n";

void USART1_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    USART_InitTypeDef USART_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_USART1, ENABLE);

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    USART_InitStructure.USART_BaudRate = 115200;
    USART_InitStructure.USART_WordLength = USART_WordLength_8b;
    USART_InitStructure.USART_StopBits = USART_StopBits_1;
    USART_InitStructure.USART_Parity = USART_Parity_No;
    USART_InitStructure.USART_Mode = USART_Mode_Tx | USART_Mode_Rx;
    USART_InitStructure.USART_HardwareFlowControl = USART_HardwareFlowControl_None;
    USART_Init(USART1, &USART_InitStructure);

    USART_Cmd(USART1, ENABLE);
}

void DMA_USART_TX_Init(void)
{
    DMA_InitTypeDef DMA_InitStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

    // 开启 DMA 时钟
    RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);

    // DMA 配置（内存 → USART1_DR）
    DMA_DeInit(DMA1_Channel4);  // USART1_TX 使用通道4

    DMA_InitStructure.DMA_PeripheralBaseAddr = (uint32_t)&USART1->DR;
    DMA_InitStructure.DMA_MemoryBaseAddr = (uint32_t)tx_buffer;
    DMA_InitStructure.DMA_DIR = DMA_DIR_PeripheralDST;  // 内存 → 外设
    DMA_InitStructure.DMA_BufferSize = sizeof(tx_buffer) - 1;
    DMA_InitStructure.DMA_PeripheralInc = DMA_PeripheralInc_Disable;
    DMA_InitStructure.DMA_MemoryInc = DMA_MemoryInc_Enable;
    DMA_InitStructure.DMA_PeripheralDataSize = DMA_PeripheralDataSize_Byte;
    DMA_InitStructure.DMA_MemoryDataSize = DMA_MemoryDataSize_Byte;
    DMA_InitStructure.DMA_Mode = DMA_Mode_Normal;
    DMA_InitStructure.DMA_Priority = DMA_Priority_High;
    DMA_InitStructure.DMA_M2M = DMA_M2M_Disable;
    DMA_Init(DMA1_Channel4, &DMA_InitStructure);

    // 使能 DMA 传输完成中断
    DMA_ITConfig(DMA1_Channel4, DMA_IT_TC, ENABLE);

    // NVIC 配置
    NVIC_InitStructure.NVIC_IRQChannel = DMA1_Channel4_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    // 使能 USART DMA 发送请求
    USART_DMACmd(USART1, USART_DMAReq_Tx, ENABLE);
}

volatile uint8_t dma_tx_complete = 0;

void DMA1_Channel4_IRQHandler(void)
{
    if (DMA_GetITStatus(DMA1_IT_TC4))
    {
        DMA_ClearITPendingBit(DMA1_IT_TC4);
        dma_tx_complete = 1;
    }
}

void DMA_Send(void)
{
    dma_tx_complete = 0;
    DMA_Cmd(DMA1_Channel4, DISABLE);
    DMA_SetCurrDataCounter(DMA1_Channel4, sizeof(tx_buffer) - 1);
    DMA_Cmd(DMA1_Channel4, ENABLE);
}

int main(void)
{
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    USART1_Init();
    DMA_USART_TX_Init();

    while (1)
    {
        DMA_Send();
        while (!dma_tx_complete);

        // 等待一段时间
        for (uint32_t i = 0; i < 8000000; i++);
    }
}
```

### 5.2 实验二：DMA 接收

```c
#include "stm32f10x.h"

#define RX_BUF_SIZE 64
uint8_t rx_buffer[RX_BUF_SIZE];
volatile uint8_t rx_complete = 0;
volatile uint16_t rx_len = 0;

void USART1_Init(void)
{
    // ... 同上 ...
}

void DMA_USART_RX_Init(void)
{
    DMA_InitTypeDef DMA_InitStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

    RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);

    DMA_DeInit(DMA1_Channel5);  // USART1_RX 使用通道5

    DMA_InitStructure.DMA_PeripheralBaseAddr = (uint32_t)&USART1->DR;
    DMA_InitStructure.DMA_MemoryBaseAddr = (uint32_t)rx_buffer;
    DMA_InitStructure.DMA_DIR = DMA_DIR_PeripheralSRC;  // 外设 → 内存
    DMA_InitStructure.DMA_BufferSize = RX_BUF_SIZE;
    DMA_InitStructure.DMA_PeripheralInc = DMA_PeripheralInc_Disable;
    DMA_InitStructure.DMA_MemoryInc = DMA_MemoryInc_Enable;
    DMA_InitStructure.DMA_PeripheralDataSize = DMA_PeripheralDataSize_Byte;
    DMA_InitStructure.DMA_MemoryDataSize = DMA_MemoryDataSize_Byte;
    DMA_InitStructure.DMA_Mode = DMA_Mode_Normal;
    DMA_InitStructure.DMA_Priority = DMA_Priority_High;
    DMA_InitStructure.DMA_M2M = DMA_M2M_Disable;
    DMA_Init(DMA1_Channel5, &DMA_InitStructure);

    DMA_ITConfig(DMA1_Channel5, DMA_IT_TC, ENABLE);

    NVIC_InitStructure.NVIC_IRQChannel = DMA1_Channel5_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    USART_DMACmd(USART1, USART_DMAReq_Rx, ENABLE);
    DMA_Cmd(DMA1_Channel5, ENABLE);
}

void DMA1_Channel5_IRQHandler(void)
{
    if (DMA_GetITStatus(DMA1_IT_TC5))
    {
        DMA_ClearITPendingBit(DMA1_IT_TC5);
        rx_len = RX_BUF_SIZE - DMA_GetCurrDataCounter(DMA1_Channel5);
        rx_complete = 1;

        // 重新启动 DMA
        DMA_Cmd(DMA1_Channel5, DISABLE);
        DMA_SetCurrDataCounter(DMA1_Channel5, RX_BUF_SIZE);
        DMA_Cmd(DMA1_Channel5, ENABLE);
    }
}

int main(void)
{
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    USART1_Init();
    DMA_USART_RX_Init();

    while (1)
    {
        if (rx_complete)
        {
            rx_complete = 0;
            // 处理接收到的数据
        }
    }
}
```

### 5.3 实验三：空闲中断 + DMA

```c
// 使用空闲中断检测数据包结束
// 配合 DMA 实现不定长数据接收

void USART1_IRQHandler(void)
{
    if (USART_GetITStatus(USART1, USART_IT_IDLE) != RESET)
    {
        // 清除空闲中断标志
        (void)USART1->SR;
        (void)USART1->DR;

        // 获取接收到的数据长度
        rx_len = RX_BUF_SIZE - DMA_GetCurrDataCounter(DMA1_Channel5);
        rx_complete = 1;

        // 重新启动 DMA
        DMA_Cmd(DMA1_Channel5, DISABLE);
        DMA_SetCurrDataCounter(DMA1_Channel5, RX_BUF_SIZE);
        DMA_Cmd(DMA1_Channel5, ENABLE);
    }
}
```

---

## 六、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【DMA 优势】解放 CPU，提高效率                     │
│  【USART1 TX】DMA1 通道4                            │
│  【USART1 RX】DMA1 通道5                            │
│  【传输方向】内存 → 外设 或 外设 → 内存            │
│  【空闲中断】检测数据包结束，适合不定长接收         │
└─────────────────────────────────────────────────────┘
```

### 记忆口诀

```
DMA 搬运不用 CPU，设置地址和长度。
串口发送用通道四，串口接收用通道五。
空闲中断配 DMA，不定长接收最方便。
```

---

*上一篇：Day 12 - USART 串口基础*  
*下一篇：Day 14 - 串口应用实战*