# Day 15 - I2C 协议基础学习笔记

> 学习日期：2026-07-31

---

## 一、核心知识点

### 1.1 什么是 I2C？

#### 定义

I2C（Inter-Integrated Circuit）是一种**两线式串行通信协议**，由 Philips 公司开发。

```
I2C 总线特点：
    - 只需两根线：SDA（数据线）和 SCL（时钟线）
    - 支持多主多从架构
    - 每个设备有唯一地址
    - 同步通信，半双工

典型应用：
    - EEPROM 存储（如 AT24C02）
    - 温湿度传感器（如 AHT10）
    - OLED 显示屏
    - RTC 时钟（如 DS3231）
    - IMU 传感器（如 MPU6050）
```

### 1.2 I2C 总线拓扑

```
        VCC (3.3V)
          │
    ┌─────┴─────┐
    │           │
    R           R  ← 上拉电阻（通常 4.7kΩ）
    │           │
    ├───────────┼───────────────────┐
    │           │                   │
   SDA         SCL                 ...
    │           │                   │
┌───┴───┐   ┌───┴───┐          ┌───┴───┐
│ Master│   │ Slave │          │ Slave │
│(STM32)│   │(设备1)│          │(设备2)│
│ Addr: │   │Addr:..│          │Addr:..│
└───────┘   └───────┘          └───────┘

要点：
    1. SDA 和 SCL 必须接上拉电阻
    2. 总线上可以有多个主设备和从设备
    3. 每个从设备有唯一的 7 位地址
```

### 1.3 I2C 通信时序

```
完整传输过程：

        ┌─┐   ┌─────────────────────────┐   ┌──
 START: │ │   │                         │   │
────┘   └───┘                         └───┘  STOP
          │←  地址 + 读/写 位 →│← 数据 →│
          
详细时序：

START 条件：SCL 高时，SDA 下降沿
        ___
SDA ___|   │←──── START
SCL ───────┐
           │

STOP 条件：SCL 高时，SDA 上升沿
        ___
SDA     |   │___←── STOP
SCL ───┘

数据传输：SCL 高时 SDA 必须稳定
        ┌─┐ ┌─┐ ┌─┐
SDA ───┘ └─┘ └─┘ └───
        │   │   │
SCL ───┘ └─┘ └─┘ └───
        ←── 数据有效
```

### 1.4 I2C 地址

```
7 位地址 + 1 位读/写：

┌───────────────────────┬─────┐
│   7 位从机地址        │ R/W │
├───────────────────────┼─────┤
│ A6 A5 A4 A3 A2 A1 A0  │  0  │ ← 写
│ A6 A5 A4 A3 A2 A1 A0  │  1  │ ← 读
└───────────────────────┴─────┘

常用设备地址：
    AT24C02 EEPROM：0x50
    AHT10 温湿度：  0x38
    OLED SSD1306：  0x3C（写）/0x3D（读）
    MPU6050：       0x68
    DS3231 RTC：    0x68
```

---

## 二、常用库函数

### 2.1 I2C 初始化

```c
// 开启 I2C 时钟
RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C1, ENABLE);

// I2C 初始化
void I2C_Init(I2C_TypeDef* I2Cx, I2C_InitTypeDef* I2C_InitStruct);

// 结构体定义
typedef struct {
    uint32_t I2C_ClockSpeed;      // 时钟速度（最高 400kHz）
    uint16_t I2C_Mode;            // 模式
    uint16_t I2C_DutyCycle;       // 占空比
    uint16_t I2C_OwnAddress1;     // 自身地址
    uint16_t I2c_Ack;             // 应答使能
    uint16_t I2C_AcknowledgedAddress; // 应答地址位数
} I2C_InitTypeDef;

// 使能 I2C
void I2C_Cmd(I2C_TypeDef* I2Cx, FunctionalState NewState);
```

### 2.2 I2C 通信函数

```c
// 产生 START 条件
void I2C_GenerateSTART(I2C_TypeDef* I2Cx, FunctionalState NewState);

// 产生 STOP 条件
void I2C_GenerateSTOP(I2C_TypeDef* I2Cx, FunctionalState NewState);

// 发送地址
void I2C_Send7bitAddress(I2C_TypeDef* I2Cx, uint8_t Address, uint8_t I2C_Direction);
    // I2C_Direction_Transmitter（写）
    // I2C_Direction_Receiver（读）

// 发送数据
void I2C_SendData(I2C_TypeDef* I2Cx, uint8_t Data);

// 接收数据
uint8_t I2C_ReceiveData(I2C_TypeDef* I2Cx);

// 等待事件
ErrorStatus I2C_CheckEvent(I2C_TypeDef* I2Cx, uint32_t I2C_EVENT);

// 常用事件
I2C_EVENT_MASTER_MODE_SELECT          // SB=1，主模式已选择
I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED  // 地址发送成功
I2C_EVENT_MASTER_RECEIVER_MODE_SELECTED     // 地址接收成功
I2C_EVENT_MASTER_BYTE_TRANSMITTING    // 数据发送中
I2C_EVENT_MASTER_BYTE_TRANSMITTED     // 数据发送完成
I2C_EVENT_MASTER_BYTE_RECEIVED        // 数据接收完成

// 应答配置
void I2C_AcknowledgeConfig(I2C_TypeDef* I2Cx, FunctionalState NewState);
```

---

## 三、核心原理

### 3.1 写数据流程

```
写入一个字节到指定地址：

1. 产生 START
2. 发送设备地址 + 写位（等待 ACK）
3. 发送寄存器地址（等待 ACK）
4. 发送数据（等待 ACK）
5. 产生 STOP

代码流程：
    I2C_GenerateSTART(I2C1, ENABLE);
    I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_MODE_SELECT);
    
    I2C_Send7bitAddress(I2C1, 0x50, I2C_Direction_Transmitter);
    I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED);
    
    I2C_SendData(I2C1, reg_addr);
    I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_BYTE_TRANSMITTED);
    
    I2C_SendData(I2C1, data);
    I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_BYTE_TRANSMITTED);
    
    I2C_GenerateSTOP(I2C1, ENABLE);
```

### 3.2 读数据流程

```
从指定地址读取一个字节：

1. 产生 START
2. 发送设备地址 + 写位
3. 发送寄存器地址
4. 产生 RESTART
5. 发送设备地址 + 读位
6. 接收数据
7. 产生 STOP

代码流程：
    // 写入要读的寄存器地址
    I2C_GenerateSTART(I2C1, ENABLE);
    I2C_Send7bitAddress(I2C1, 0x50, I2C_Direction_Transmitter);
    I2C_SendData(I2C1, reg_addr);
    
    // RESTART + 读
    I2C_GenerateSTART(I2C1, ENABLE);
    I2C_Send7bitAddress(I2C1, 0x50, I2C_Direction_Receiver);
    
    // 禁用 ACK（只读一个字节）
    I2C_AcknowledgeConfig(I2C1, DISABLE);
    
    // 读取数据
    I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_BYTE_RECEIVED);
    data = I2C_ReceiveData(I2C1);
    
    I2C_GenerateSTOP(I2C1, ENABLE);
```

---

## 四、电路连接

### 4.1 硬件清单

| 组件 | 数量 | 说明 |
|------|------|------|
| STM32F103C8T6 | 1 | - |
| I2C 设备 | 1 | 如 AT24C02 EEPROM |
| 上拉电阻 | 2 | 4.7kΩ |

### 4.2 推荐接线

| STM32 引脚 | 连接 | 说明 |
|------------|------|------|
| PB6 | SCL | I2C1 时钟线 |
| PB7 | SDA | I2C1 数据线 |
| 3.3V | VCC | 设备电源 |
| GND | GND | 共地 |

### 4.3 电路图

```
        STM32F103C8T6              I2C 设备
       ┌─────────────┐            ┌─────────┐
       │             │            │         │
       │    PB6 ─────┼────┬───────┼─ SCL    │
       │    (SCL)    │    │       │         │
       │             │    R       │         │
       │             │    │       │         │
       │    PB7 ─────┼────┼───┬───┼─ SDA    │
       │    (SDA)    │    │   │   │         │
       │             │    R   │   │         │
       │             │    │   │   │         │
       │    3.3V ────┼────┴───┼───┼─ VCC    │
       │             │        │   │         │
       │    GND ─────┼────────┴───┼─ GND    │
       │             │            │         │
       └─────────────┘            └─────────┘

上拉电阻 R = 4.7kΩ
```

---

## 五、完整代码实现

### 5.1 实验一：I2C 初始化

```c
#include "stm32f10x.h"

void I2C1_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    I2C_InitTypeDef I2C_InitStructure;

    // 开启时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C1, ENABLE);

    // 配置 GPIO（复用开漏输出）
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6 | GPIO_Pin_7;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_OD;  // 复用开漏
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOB, &GPIO_InitStructure);

    // I2C 配置
    I2C_InitStructure.I2C_ClockSpeed = 100000;  // 100kHz
    I2C_InitStructure.I2C_Mode = I2C_Mode_I2C;
    I2C_InitStructure.I2C_DutyCycle = I2C_DutyCycle_2;
    I2C_InitStructure.I2C_OwnAddress1 = 0x00;
    I2C_InitStructure.I2C_Ack = I2C_Ack_Enable;
    I2C_InitStructure.I2C_AcknowledgedAddress = I2C_AcknowledgedAddress_7bit;
    I2C_Init(I2C1, &I2C_InitStructure);

    // 使能 I2C
    I2C_Cmd(I2C1, ENABLE);
}
```

### 5.2 实验二：I2C 写字节

```c
void I2C_WriteByte(uint8_t addr, uint8_t reg, uint8_t data)
{
    // 产生 START
    I2C_GenerateSTART(I2C1, ENABLE);
    while (!I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_MODE_SELECT));

    // 发送设备地址 + 写位
    I2C_Send7bitAddress(I2C1, addr << 1, I2C_Direction_Transmitter);
    while (!I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED));

    // 发送寄存器地址
    I2C_SendData(I2C1, reg);
    while (!I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_BYTE_TRANSMITTED));

    // 发送数据
    I2C_SendData(I2C1, data);
    while (!I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_BYTE_TRANSMITTED));

    // 产生 STOP
    I2C_GenerateSTOP(I2C1, ENABLE);
}
```

### 5.3 实验三：I2C 读字节

```c
uint8_t I2C_ReadByte(uint8_t addr, uint8_t reg)
{
    uint8_t data;

    // 写入寄存器地址
    I2C_GenerateSTART(I2C1, ENABLE);
    while (!I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_MODE_SELECT));

    I2C_Send7bitAddress(I2C1, addr << 1, I2C_Direction_Transmitter);
    while (!I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED));

    I2C_SendData(I2C1, reg);
    while (!I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_BYTE_TRANSMITTED));

    // RESTART + 读
    I2C_GenerateSTART(I2C1, ENABLE);
    while (!I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_MODE_SELECT));

    I2C_Send7bitAddress(I2C1, addr << 1, I2C_Direction_Receiver);
    while (!I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_RECEIVER_MODE_SELECTED));

    // 禁用 ACK（只读一个字节）
    I2C_AcknowledgeConfig(I2C1, DISABLE);

    // 读取数据
    while (!I2C_CheckEvent(I2C1, I2C_EVENT_MASTER_BYTE_RECEIVED));
    data = I2C_ReceiveData(I2C1);

    // 产生 STOP
    I2C_GenerateSTOP(I2C1, ENABLE);

    // 重新使能 ACK
    I2C_AcknowledgeConfig(I2C1, ENABLE);

    return data;
}
```

---

## 六、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【I2C 线路】SDA（数据）+ SCL（时钟）              │
│  【上拉电阻】必须接，通常 4.7kΩ                    │
│  【地址格式】7 位地址 + 1 位读/写                  │
│  【GPIO 模式】复用开漏输出                          │
│  【写流程】START → 地址+W → 寄存器 → 数据 → STOP  │
│  【读流程】START → 地址+W → 寄存器 → RESTART →    │
│           地址+R → 数据 → STOP                     │
└─────────────────────────────────────────────────────┘
```

### 记忆口诀

```
I2C 两线走天下，数据时钟分开挂。
上拉电阻不能少，地址七位记得牢。
先发地址再发数，ACK 应答要等到。
```

---

*上一篇：Day 14 - 串口应用实战*  
*下一篇：Day 16 - I2C OLED 显示屏*