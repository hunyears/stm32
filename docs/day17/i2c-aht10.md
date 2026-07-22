# Day 17 - I2C 温湿度传感器学习笔记

> 学习日期：2026-08-02

---

## 一、核心知识点

### 1.1 AHT10 温湿度传感器

#### 简介

AHT10 是一款高精度 I2C 温湿度传感器芯片。

```
特点：
    - 温度精度：±0.3°C
    - 湿度精度：±2% RH
    - I2C 接口
    - 工作电压：2.0V ~ 5.5V
    - I2C 地址：0x38

内部结构：
    ┌─────────────────────────────┐
    │  温度传感器  │  湿度传感器  │
    │      │            │        │
    │      └────┬───────┘        │
    │           ↓                │
    │    ADC 转换模块             │
    │           │                │
    │    校准系数存储             │
    │           │                │
    │    I2C 接口                │
    └─────────────────────────────┘
```

### 1.2 数据格式

```
AHT10 数据格式（7 字节）：

字节0：状态字节
    Bit7：Busy（1=忙碌，0=空闲）
    Bit[6:3]：保留
    Bit[2:0]：校准状态

字节1-2：湿度数据（20位）
    Byte1：湿度[19:12]
    Byte2：湿度[11:4]
    Byte3 高4位：湿度[3:0]

字节3-5：温度数据（20位）
    Byte3 低4位：温度[19:16]
    Byte4：温度[15:8]
    Byte5：温度[7:0]

字节6：CRC 校验

计算公式：
    湿度(%) = (湿度原始值 / 2^20) × 100
    温度(°C) = (温度原始值 / 2^20) × 200 - 50
```

---

## 二、常用命令

```c
#define AHT10_ADDR         0x38

// 命令定义
#define AHT10_INIT_CMD     0xE1  // 初始化
#define AHT10_TRIGGER_CMD  0xAC  // 触发测量
#define AHT10_SOFTRESET    0xBA  // 软复位

// 参数
#define AHT10_DATA_NOP     0x00
#define AHT10_DATA_MSB     0x33  // 使能 CRC
```

---

## 三、完整代码实现

### 3.1 AHT10 初始化

```c
void AHT10_Init(void)
{
    I2C_WriteByte(AHT10_ADDR, 0xE1, 0x08);  // 校准使能
    I2C_WriteByte(AHT10_ADDR, 0xE1, 0x00);
    delay_ms(40);
}
```

### 3.2 读取温湿度

```c
void AHT10_Read(float *temp, float *humi)
{
    uint8_t data[7];
    uint32_t raw_humi, raw_temp;

    // 触发测量
    I2C_WriteByte(AHT10_ADDR, 0xAC, 0x33);
    I2C_WriteByte(AHT10_ADDR, 0xAC, 0x00);
    delay_ms(80);

    // 读取 7 字节数据
    I2C_ReadBytes(AHT10_ADDR, data, 7);

    // 计算湿度
    raw_humi = ((uint32_t)data[1] << 12) |
               ((uint32_t)data[2] << 4) |
               (data[3] >> 4);
    *humi = (float)raw_humi / 1048576.0f * 100.0f;

    // 计算温度
    raw_temp = ((uint32_t)(data[3] & 0x0F) << 16) |
               ((uint32_t)data[4] << 8) |
               data[5];
    *temp = (float)raw_temp / 1048576.0f * 200.0f - 50.0f;
}
```

---

## 四、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【AHT10 地址】0x38                                 │
│  【测量流程】触发 → 等待 → 读取                     │
│  【数据格式】7 字节（状态+湿度+温度+CRC）           │
│  【温度计算】raw / 2^20 × 200 - 50                  │
│  【湿度计算】raw / 2^20 × 100                       │
└─────────────────────────────────────────────────────┘
```

---

*上一篇：Day 16 - I2C OLED 显示屏*  
*下一篇：Day 18 - SPI 协议基础*