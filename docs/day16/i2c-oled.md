# Day 16 - I2C OLED 显示屏学习笔记

> 学习日期：2026-07-31

---

## 一、核心知识点

### 1.1 OLED 显示屏简介

#### SSD1306 芯片

SSD1306 是一款单片 CMOS OLED/PLED 驱动芯片，支持 128x64 点阵显示。

```
OLED 模块特点：
    - 分辨率：128×64 像素
    - 接口：I2C（4 线）或 SPI（7 线）
    - 工作电压：3.3V ~ 5V
    - I2C 地址：0x3C（默认）

显示原理：
    - 每个像素点是一个 OLED 发光单元
    - 通过控制行列电极点亮/熄灭像素
    - 显存（GDDRAM）映射到屏幕显示
```

### 1.2 显存结构

```
SSD1306 显存映射（128×64）：

    ┌────────────────────────────────────────┐
    │  PAGE0  │ 列0 列1 ... 列127           │
    │  PAGE1  │ ...                         │
    │  PAGE2  │ ...                         │
    │  PAGE3  │ ...                         │
    │  PAGE4  │ ...                         │
    │  PAGE5  │ ...                         │
    │  PAGE6  │ ...                         │
    │  PAGE7  │ ...                         │
    └────────────────────────────────────────┘

    - 显存分为 8 页（PAGE0~PAGE7）
    - 每页 128 列，每列 8 位（对应 8 行像素）
    - 总计 128×64 = 8192 位 = 1024 字节

写入方式：
    - 先写入页地址
    - 再写入列地址
    - 然后连续写入数据
```

### 1.3 I2C 写命令与写数据

```
I2C 写入格式：

写命令：
    START → 地址(0x78) → CONTROL(0x00) → 命令 → STOP

写数据：
    START → 地址(0x78) → CONTROL(0x40) → 数据 → STOP

CONTROL 字节：
    0x00 = 后续是命令
    0x40 = 后续是数据
```

---

## 二、常用库函数

### 2.1 SSD1306 基本命令

```c
// 基本命令
#define SSD1306_DISPLAYOFF        0xAE  // 关闭显示
#define SSD1306_DISPLAYON         0xAF  // 开启显示
#define SSD1306_SETDISPLAYCLOCKDIV 0xD5 // 设置时钟分频
#define SSD1306_SETMULTIPLEX      0xA8  // 设置多路复用率
#define SSD1306_SETDISPLAYOFFSET  0xD3  // 设置显示偏移
#define SSD1306_SETSTARTLINE      0x40  // 设置起始行
#define SSD1306_MEMORYMODE        0x20  // 设置内存模式
#define SSD1306_SEGREMAP          0xA0  // 段重映射
#define SSD1306_COMSCANINC        0xC0  // COM 扫描方向
#define SSD1306_SETCOMPINS        0xDA  // 设置 COM 引脚
#define SSD1306_SETCONTRAST       0x81  // 设置对比度
#define SSD1306_SETPRECHARGE      0xD9  // 设置预充电
#define SSD1306_SETVCOMDETECT     0xDB  // 设置 VCOM
#define SSD1306_CHARGEPUMP        0x8D  // 电荷泵设置
```

---

## 三、核心原理

### 3.1 初始化序列

```c
// OLED 初始化序列
void OLED_Init(void)
{
    OLED_WriteCmd(0xAE);  // 关闭显示
    
    OLED_WriteCmd(0xD5);  // 设置时钟分频
    OLED_WriteCmd(0x80);  // 建议值
    
    OLED_WriteCmd(0xA8);  // 设置多路复用率
    OLED_WriteCmd(0x3F);  // 1/64 duty
    
    OLED_WriteCmd(0xD3);  // 设置显示偏移
    OLED_WriteCmd(0x00);  // 无偏移
    
    OLED_WriteCmd(0x40);  // 设置起始行
    
    OLED_WriteCmd(0x8D);  // 电荷泵设置
    OLED_WriteCmd(0x14);  // 开启电荷泵
    
    OLED_WriteCmd(0x20);  // 内存模式
    OLED_WriteCmd(0x02);  // 页地址模式
    
    OLED_WriteCmd(0xA1);  // 段重映射
    OLED_WriteCmd(0xC8);  // COM 扫描方向
    
    OLED_WriteCmd(0xDA);  // COM 引脚配置
    OLED_WriteCmd(0x12);
    
    OLED_WriteCmd(0x81);  // 对比度
    OLED_WriteCmd(0xCF);
    
    OLED_WriteCmd(0xD9);  // 预充电
    OLED_WriteCmd(0xF1);
    
    OLED_WriteCmd(0xDB);  // VCOM
    OLED_WriteCmd(0x40);
    
    OLED_WriteCmd(0xA4);  // 全屏显示关闭
    OLED_WriteCmd(0xA6);  // 正常显示
    
    OLED_WriteCmd(0xAF);  // 开启显示
}
```

### 3.2 字符显示原理

```
ASCII 字符（8×16 像素）：

    'A' 的点阵数据：
    
    0x00, 0x00, 0xC0, 0x38, 0xE0, 0x00, 0x00, 0x00,
    0x20, 0x3C, 0x23, 0x02, 0x02, 0x27, 0x38, 0x20

    对应显示：
         ████
       ██    ██
       ██    ██
       ████████
       ██    ██
       ██    ██
       ██    ██
       ██    ██

显示流程：
    1. 定位光标位置（页地址 + 列地址）
    2. 查表获取字符点阵数据
    3. 连续写入显存
```

---

## 四、电路连接

### 4.1 硬件清单

| 组件 | 数量 | 说明 |
|------|------|------|
| STM32F103C8T6 | 1 | - |
| OLED 0.96寸 | 1 | SSD1306，I2C 接口 |

### 4.2 推荐接线

| OLED 引脚 | STM32 引脚 | 说明 |
|-----------|------------|------|
| VCC | 3.3V | 电源 |
| GND | GND | 地 |
| SCL | PB6 | I2C 时钟 |
| SDA | PB7 | I2C 数据 |

---

## 五、完整代码实现

### 5.1 实验一：OLED 初始化与清屏

```c
#include "stm32f10x.h"

#define OLED_ADDR 0x78

void I2C1_Init(void);
void OLED_WriteCmd(uint8_t cmd);
void OLED_WriteData(uint8_t data);
void OLED_Init(void);
void OLED_Clear(void);
void OLED_SetPos(uint8_t page, uint8_t col);

int main(void)
{
    I2C1_Init();
    OLED_Init();
    OLED_Clear();
    
    while (1);
}
```

### 5.2 实验二：显示字符和字符串

```c
// 8x16 ASCII 字库（部分）
const uint8_t F8X16[] = {
    // 'A'
    0x00, 0x00, 0xC0, 0x38, 0xE0, 0x00, 0x00, 0x00,
    0x20, 0x3C, 0x23, 0x02, 0x02, 0x27, 0x38, 0x20,
    // ... 更多字符
};

void OLED_ShowChar(uint8_t x, uint8_t y, char ch)
{
    uint8_t c = ch - ' ';
    uint8_t i;
    
    OLED_SetPos(y, x);
    for (i = 0; i < 8; i++)
        OLED_WriteData(F8X16[c * 16 + i]);
    
    OLED_SetPos(y + 1, x);
    for (i = 0; i < 8; i++)
        OLED_WriteData(F8X16[c * 16 + i + 8]);
}

void OLED_ShowString(uint8_t x, uint8_t y, char *str)
{
    while (*str)
    {
        OLED_ShowChar(x, y, *str++);
        x += 8;
        if (x > 120) { x = 0; y += 2; }
    }
}
```

---

## 六、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【OLED 地址】I2C 写地址 0x78                       │
│  【显存结构】8 页 × 128 列                          │
│  【写命令】地址 + 0x00 + 命令                       │
│  【写数据】地址 + 0x40 + 数据                       │
│  【初始化】按序发送配置命令                         │
└─────────────────────────────────────────────────────┘
```

---

*上一篇：Day 15 - I2C 协议基础*  
*下一篇：Day 17 - I2C 温湿度传感器*