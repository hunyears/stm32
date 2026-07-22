# Day 19 - SPI Flash 存储学习笔记

> 学习日期：2026-08-04

---

## 一、核心知识点

### 1.1 W25Q64 Flash 芯片

#### 简介

W25Q64 是一款 64Mbit（8MB）的 SPI Flash 存储芯片。

```
特点：
    - 容量：8MB（64Mbit）
    - 接口：SPI（支持标准/Dual/Quad SPI）
    - 时钟：最高 133MHz
    - 擦写寿命：10万次
    - 数据保持：20年

存储结构：
    - 8MB = 8192 KB
    - 分为 128 个块（Block，64KB）
    - 每块分为 16 个扇区（Sector，4KB）
    - 每扇区分为 16 页（Page，256字节）

最小操作单位：
    - 写入：页（256字节）
    - 擦除：扇区（4KB）/ 块（64KB）/ 全片
```

### 1.2 Flash 操作原理

```
Flash 特性：
    1. 写入前必须先擦除
    2. 擦除后所有位变为 1
    3. 写入只能将 1 变为 0
    4. 不能直接将 0 写为 1

操作流程：
    写入数据：
        1. 检查目标区域是否已擦除
        2. 若有数据，需先擦除
        3. 发送写使能命令
        4. 发送页编程命令
        5. 写入数据

    擦除扇区：
        1. 发送写使能命令
        2. 发送扇区擦除命令
        3. 等待擦除完成
```

---

## 二、常用命令

```c
// W25Q64 命令定义
#define W25X_WriteEnable        0x06
#define W25X_WriteDisable       0x04
#define W25X_ReadStatusReg1     0x05
#define W25X_ReadStatusReg2     0x35
#define W25X_WriteStatusReg     0x01
#define W25X_PageProgram        0x02
#define W25X_SectorErase        0x20
#define W25X_BlockErase         0xD8
#define W25X_ChipErase          0xC7
#define W25X_ReadData           0x03
#define W25X_JedecDeviceID      0x9F
#define W25X_PowerDown          0xB9
```

---

## 三、核心原理

### 3.1 状态寄存器

```
状态寄存器 1（S1）：
    Bit 0：BUSY（忙碌标志）
    Bit 1：WEL（写使能标志）
    Bit [7:2]：保护位

检测忙碌状态：
    while (W25Q64_ReadSR() & 0x01);  // 等待空闲
```

---

## 四、完整代码实现

### 4.1 读取 JEDEC ID

```c
uint32_t W25Q64_ReadID(void)
{
    uint32_t id;

    W25Q64_CS_LOW();
    SPI_ReadWriteByte(W25X_JedecDeviceID);
    id = SPI_ReadWriteByte(0xFF) << 16;  // Manufacturer
    id |= SPI_ReadWriteByte(0xFF) << 8;   // Device ID High
    id |= SPI_ReadWriteByte(0xFF);        // Device ID Low
    W25Q64_CS_HIGH();

    return id;
}
```

### 4.2 读取数据

```c
void W25Q64_Read(uint8_t *buf, uint32_t addr, uint16_t len)
{
    W25Q64_CS_LOW();
    SPI_ReadWriteByte(W25X_ReadData);
    SPI_ReadWriteByte((addr >> 16) & 0xFF);
    SPI_ReadWriteByte((addr >> 8) & 0xFF);
    SPI_ReadWriteByte(addr & 0xFF);

    while (len--)
        *buf++ = SPI_ReadWriteByte(0xFF);

    W25Q64_CS_HIGH();
}
```

### 4.3 页写入

```c
void W25Q64_WritePage(uint8_t *buf, uint32_t addr, uint16_t len)
{
    W25Q64_WriteEnable();

    W25Q64_CS_LOW();
    SPI_ReadWriteByte(W25X_PageProgram);
    SPI_ReadWriteByte((addr >> 16) & 0xFF);
    SPI_ReadWriteByte((addr >> 8) & 0xFF);
    SPI_ReadWriteByte(addr & 0xFF);

    while (len--)
        SPI_ReadWriteByte(*buf++);

    W25Q64_CS_HIGH();

    W25Q64_WaitBusy();
}
```

### 4.4 扇区擦除

```c
void W25Q64_EraseSector(uint32_t addr)
{
    W25Q64_WriteEnable();

    W25Q64_CS_LOW();
    SPI_ReadWriteByte(W25X_SectorErase);
    SPI_ReadWriteByte((addr >> 16) & 0xFF);
    SPI_ReadWriteByte((addr >> 8) & 0xFF);
    SPI_ReadWriteByte(addr & 0xFF);
    W25Q64_CS_HIGH();

    W25Q64_WaitBusy();
}
```

---

## 五、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【Flash 特性】写入前必须先擦除                     │
│  【页大小】256 字节                                 │
│  【扇区大小】4KB                                    │
│  【状态检测】读取状态寄存器 Bit0 判断忙碌          │
│  【写使能】每次写入/擦除前必须发送 0x06            │
└─────────────────────────────────────────────────────┘
```

---

*上一篇：Day 18 - SPI 协议基础*  
*下一篇：Day 20 - ADC 多通道采集*