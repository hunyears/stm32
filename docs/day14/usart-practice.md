# Day 14 - 串口应用实战学习笔记

> 学习日期：2026-07-30

---

## 一、核心知识点

### 1.1 串口应用场景

```
┌─────────────────────────────────────────────────────┐
│  串口常见应用                                       │
├─────────────────────────────────────────────────────┤
│  调试输出    │ printf 打印调试信息                 │
│  命令控制    │ 通过串口发送命令控制设备            │
│  数据传输    │ 传感器数据上传、配置参数下发        │
│  固件升级    │ 通过串口烧录固件（ISP）             │
│  模块通信    │ 与蓝牙、WiFi、GPS 等模块通信        │
└─────────────────────────────────────────────────────┘
```

### 1.2 数据协议设计

```
简单文本协议示例：

命令格式：
    COMMAND [参数] [参数] ...
    
示例：
    LED ON       - 打开 LED
    LED OFF      - 关闭 LED
    PWM 50       - 设置 PWM 占空比为 50%
    READ TEMP    - 读取温度

二进制协议示例：

帧结构：
    ┌────┬────┬────┬──────┬────┐
    │ 头 │ 命令│ 长度│ 数据 │ 校验│
    │ 0xAA│ CMD │ LEN │ DATA │ CRC │
    └────┴────┴────┴──────┴────┘
```

---

## 二、常用库函数

与 Day 12、Day 13 相同。

---

## 三、完整代码实现

### 3.1 实验一：串口命令解析器

```c
#include "stm32f10x.h"
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#define RX_BUF_SIZE 128
char rx_buffer[RX_BUF_SIZE];
volatile uint8_t rx_index = 0;
volatile uint8_t cmd_ready = 0;

void USART1_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    USART_InitTypeDef USART_InitStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

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

    USART_ITConfig(USART1, USART_IT_RXNE, ENABLE);

    NVIC_InitStructure.NVIC_IRQChannel = USART1_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    USART_Cmd(USART1, ENABLE);
}

void USART1_IRQHandler(void)
{
    char c;
    if (USART_GetITStatus(USART1, USART_IT_RXNE) != RESET)
    {
        c = USART_ReceiveData(USART1);

        if (c == '\r' || c == '\n')
        {
            if (rx_index > 0)
            {
                rx_buffer[rx_index] = '\0';
                cmd_ready = 1;
            }
        }
        else if (rx_index < RX_BUF_SIZE - 1)
        {
            rx_buffer[rx_index++] = c;
        }
    }
}

int fputc(int ch, FILE *f)
{
    USART_SendData(USART1, ch);
    while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
    return ch;
}

// LED 控制
void LED_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOC, &GPIO_InitStructure);
}

void LED_On(void)  { GPIO_ResetBits(GPIOC, GPIO_Pin_13); }
void LED_Off(void) { GPIO_SetBits(GPIOC, GPIO_Pin_13); }
void LED_Toggle(void)
{
    GPIO_WriteBit(GPIOC, GPIO_Pin_13,
        (BitAction)(1 - GPIO_ReadOutputDataBit(GPIOC, GPIO_Pin_13)));
}

// 命令解析
void Parse_Command(char *cmd)
{
    char *token;

    token = strtok(cmd, " ");

    if (strcmp(token, "LED") == 0)
    {
        token = strtok(NULL, " ");
        if (strcmp(token, "ON") == 0)
        {
            LED_On();
            printf("LED turned ON\r\n");
        }
        else if (strcmp(token, "OFF") == 0)
        {
            LED_Off();
            printf("LED turned OFF\r\n");
        }
        else if (strcmp(token, "TOGGLE") == 0)
        {
            LED_Toggle();
            printf("LED toggled\r\n");
        }
        else
        {
            printf("Unknown LED command: %s\r\n", token);
        }
    }
    else if (strcmp(token, "HELP") == 0)
    {
        printf("Available commands:\r\n");
        printf("  LED ON     - Turn on LED\r\n");
        printf("  LED OFF    - Turn off LED\r\n");
        printf("  LED TOGGLE - Toggle LED\r\n");
        printf("  HELP       - Show this help\r\n");
    }
    else
    {
        printf("Unknown command: %s\r\n", token);
    }
}

int main(void)
{
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    LED_Init();
    USART1_Init();

    printf("\r\n========================================\r\n");
    printf("  STM32 Serial Command Demo\r\n");
    printf("  Type HELP for available commands\r\n");
    printf("========================================\r\n");

    while (1)
    {
        if (cmd_ready)
        {
            cmd_ready = 0;
            printf("\r\n> %s\r\n", rx_buffer);
            Parse_Command(rx_buffer);
            rx_index = 0;
        }
    }
}
```

### 3.2 实验二：传感器数据上报

```c
// 定时读取传感器数据并通过串口上报

#include "stm32f10x.h"
#include <stdio.h>

volatile uint8_t report_flag = 0;

void TIM2_Init(void)
{
    TIM_TimeBaseInitTypeDef TIM_TimeBaseStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);

    TIM_TimeBaseStructure.TIM_Period = 9999;
    TIM_TimeBaseStructure.TIM_Prescaler = 7199;
    TIM_TimeBaseStructure.TIM_ClockDivision = 0;
    TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInit(TIM2, &TIM_TimeBaseStructure);

    TIM_ITConfig(TIM2, TIM_IT_Update, ENABLE);

    NVIC_InitStructure.NVIC_IRQChannel = TIM2_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    TIM_Cmd(TIM2, ENABLE);
}

void TIM2_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM2, TIM_IT_Update) != RESET)
    {
        report_flag = 1;
        TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
    }
}

// 模拟传感器读取
float Read_Temperature(void)
{
    return 25.0f + (rand() % 100) / 10.0f;
}

float Read_Humidity(void)
{
    return 50.0f + (rand() % 100) / 10.0f;
}

int main(void)
{
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    USART1_Init();
    TIM2_Init();

    printf("Sensor Data Reporter\r\n");

    while (1)
    {
        if (report_flag)
        {
            report_flag = 0;

            float temp = Read_Temperature();
            float humi = Read_Humidity();

            printf("{\"temp\": %.1f, \"humidity\": %.1f}\r\n", temp, humi);
        }
    }
}
```

### 3.3 实验三：二进制协议通信

```c
// 二进制帧协议
// 帧格式：0xAA | CMD | LEN | DATA[0..N] | CRC

typedef struct {
    uint8_t header;
    uint8_t cmd;
    uint8_t len;
    uint8_t data[32];
    uint8_t crc;
} Frame_TypeDef;

uint8_t Calculate_CRC(uint8_t *data, uint8_t len)
{
    uint8_t crc = 0;
    for (uint8_t i = 0; i < len; i++)
    {
        crc += data[i];
    }
    return crc;
}

void Send_Frame(uint8_t cmd, uint8_t *data, uint8_t len)
{
    uint8_t crc;

    USART_SendData(USART1, 0xAA);  // Header
    while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);

    USART_SendData(USART1, cmd);   // Command
    while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);

    USART_SendData(USART1, len);   // Length
    while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);

    for (uint8_t i = 0; i < len; i++)  // Data
    {
        USART_SendData(USART1, data[i]);
        while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
    }

    crc = Calculate_CRC(data, len);
    USART_SendData(USART1, crc);   // CRC
    while (USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);
}
```

---

## 四、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【命令解析】strtok 分割字符串，strcmp 匹配命令    │
│  【定时上报】定时器中断触发数据采集和上报          │
│  【二进制协议】帧头 + 命令 + 长度 + 数据 + 校验    │
│  【调试输出】printf 重定向到串口                    │
└─────────────────────────────────────────────────────┘
```

---

*上一篇：Day 13 - USART 中断与 DMA*  
*下一篇：Day 15 - I2C 协议基础*