# Day 18 - SPI 协议基础学习笔记

> 学习日期：2026-08-03

---

## 一、核心知识点

### 1.1 什么是 SPI？

#### 定义

SPI（Serial Peripheral Interface）是一种**四线高速同步串行通信协议**。

```
SPI 四线制：
    MOSI（Master Out Slave In） - 主机发送数据
    MISO（Master In Slave Out）  - 从机发送数据
    SCK  （Serial Clock）        - 时钟线
    CS/SS（Chip Select）         - 片选信号

特点：
    - 全双工通信
    - 高速（可达数十 MHz）
    - 同步传输
    - 主从架构
```

### 1.2 SPI 与 I2C 对比

| 特性 | SPI | I2C |
|------|-----|-----|
| 线数 | 4线 | 2线 |
| 速度 | 快（MHz级） | 慢（kHz级） |
| 模式 | 全双工 | 半双工 |
| 寻址 | 硬件片选 | 软件地址 |
| 应用 | Flash、LCD、SD卡 | 传感器、EEPROM |

### 1.3 SPI 四种工作模式

```
┌─────────────────────────────────────────────────────┐
│  模式 │ CPOL │ CPHA │ 说明                         │
├───────┼──────┼──────┼───────────────────────────────│
│  0    │  0   │  0   │ 空闲低电平，第一个边沿采样   │
│  1    │  0   │  1   │ 空闲低电平，第二个边沿采样   │
│  2    │  1   │  0   │ 空闲高电平，第一个边沿采样   │
│  3    │  1   │  1   │ 空闲高电平，第二个边沿采样   │
└─────────────────────────────────────────────────────┘

常用模式：
    Mode 0：大部分 Flash、传感器
    Mode 3：部分 LCD、SD卡
```

---

## 二、常用库函数

### 2.1 SPI 初始化

```c
void SPI_Init(SPI_TypeDef* SPIx, SPI_InitTypeDef* SPI_InitStruct);

typedef struct {
    uint16_t SPI_Direction;     // 传输方向
    uint16_t SPI_Mode;          // 主/从模式
    uint16_t SPI_DataSize;      // 数据位数
    uint16_t SPI_CPOL;          // 时钟极性
    uint16_t SPI_CPHA;          // 时钟相位
    uint16_t SPI_NSS;           // NSS 管理
    uint16_t SPI_BaudRatePrescaler; // 波特率分频
    uint16_t SPI_FirstBit;      // 传输顺序
} SPI_InitTypeDef;

// 使能 SPI
void SPI_Cmd(SPI_TypeDef* SPIx, FunctionalState NewState);
```

### 2.2 数据传输

```c
// 发送数据
void SPI_I2S_SendData(SPI_TypeDef* SPIx, uint16_t Data);

// 接收数据
uint16_t SPI_I2S_ReceiveData(SPI_TypeDef* SPIx);

// 检查标志
FlagStatus SPI_I2S_GetFlagStatus(SPI_TypeDef* SPIx, uint16_t SPI_I2S_FLAG);

// 常用标志
SPI_I2S_FLAG_TXE   // 发送缓冲区空
SPI_I2S_FLAG_RXNE  // 接收缓冲区非空
SPI_I2S_FLAG_BSY   // 忙碌标志
```

---

## 三、核心原理

### 3.1 SPI 传输过程

```
主机发送一字节：

    CS  ──┐          ┌────────
          │          │
          └──────────┘
    
    SCK  ─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐ ┌─┐
          └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘ └─┘
    
    MOSI ══╪═══╪═══╪═══╪═══╪═══╪═══╪═══╪═══
          D7  D6  D5  D4  D3  D2  D1  D0
    
    MISO ══╪═══╪═══╪═══╪═══╪═══╪═══╪═══╪═══
          D7  D6  D5  D4  D3  D2  D1  D0

流程：
    1. 拉低 CS（选中从机）
    2. 每个时钟边沿传输一位
    3. MOSI 发送，MISO 接收（同时进行）
    4. 拉高 CS（释放从机）
```

---

## 四、电路连接

### 4.1 推荐接线

| STM32 引脚 | 连接 | 说明 |
|------------|------|------|
| PA5 | SCK | SPI1 时钟 |
| PA6 | MISO | SPI1 数据输入 |
| PA7 | MOSI | SPI1 数据输出 |
| PA4 | CS | 片选（手动控制）|

---

## 五、完整代码实现

### 5.1 SPI 初始化

```c
void SPI1_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    SPI_InitTypeDef SPI_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_SPI1, ENABLE);

    // SCK, MOSI 配置
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_5 | GPIO_Pin_7;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // MISO 配置
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IN_FLOATING;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // CS 配置
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_4;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
    GPIO_SetBits(GPIOA, GPIO_Pin_4);  // 默认不选中

    // SPI 配置
    SPI_InitStructure.SPI_Direction = SPI_Direction_2Lines_FullDuplex;
    SPI_InitStructure.SPI_Mode = SPI_Mode_Master;
    SPI_InitStructure.SPI_DataSize = SPI_DataSize_8b;
    SPI_InitStructure.SPI_CPOL = SPI_CPOL_Low;
    SPI_InitStructure.SPI_CPHA = SPI_CPHA_1Edge;
    SPI_InitStructure.SPI_NSS = SPI_NSS_Soft;
    SPI_InitStructure.SPI_BaudRatePrescaler = SPI_BaudRatePrescaler_8;
    SPI_InitStructure.SPI_FirstBit = SPI_FirstBit_MSB;
    SPI_Init(SPI1, &SPI_InitStructure);

    SPI_Cmd(SPI1, ENABLE);
}
```

### 5.2 SPI 读写一字节

```c
uint8_t SPI_ReadWriteByte(uint8_t data)
{
    while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_TXE) == RESET);
    SPI_I2S_SendData(SPI1, data);

    while (SPI_I2S_GetFlagStatus(SPI1, SPI_I2S_FLAG_RXNE) == RESET);
    return SPI_I2S_ReceiveData(SPI1);
}
```

---

## 六、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【SPI 四线】MOSI、MISO、SCK、CS                    │
│  【工作模式】CPOL 和 CPHA 组合决定                 │
│  【全双工】同时发送和接收                          │
│  【GPIO 模式】SCK/MOSI 复用推挽，MISO 浮空输入     │
│  【片选控制】手动控制 CS 引脚                      │
└─────────────────────────────────────────────────────┘
```

---

*上一篇：Day 17 - I2C 温湿度传感器*  
*下一篇：Day 19 - SPI Flash 存储*