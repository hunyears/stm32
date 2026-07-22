# Day 22 - RTC 实时时钟学习笔记

> 学习日期：2026-08-07

---

## 一、核心知识点

### 1.1 RTC 简介

RTC（Real Time Clock，实时时钟）是 STM32 内置的实时时钟模块。

```
特点：
    - 独立供电（可用备用电池）
    - 32.768kHz 晶振
    - 秒、分、时、日、周、月、年
    - 闹钟功能
    - 周期性唤醒

寄存器：
    - RTC_TR：时间寄存器
    - RTC_DR：日期寄存器
    - RTC_SSR：亚秒寄存器
    - RTC_ALRMAR：闹钟 A 寄存器
```

---

## 二、常用库函数

```c
// 开启 RTC 时钟
void RCC_RTCCLKConfig(uint32_t RCC_RTCCLKSource);
void RCC_RTCCLKCmd(FunctionalState NewState);

// 等待同步
void RTC_WaitForSynchro(void);
void RTC_WaitForLastTask(void);

// 设置时间
void RTC_SetTime(uint32_t RTC_Format, RTC_TimeTypeDef* RTC_TimeStruct);
void RTC_SetDate(uint32_t RTC_Format, RTC_DateTypeDef* RTC_DateStruct);

// 读取时间
void RTC_GetTime(uint32_t RTC_Format, RTC_TimeTypeDef* RTC_TimeStruct);
void RTC_GetDate(uint32_t RTC_Format, RTC_DateTypeDef* RTC_DateStruct);
```

---

## 三、完整代码实现

### 3.1 RTC 初始化

```c
void RTC_Init(void)
{
    RTC_TimeTypeDef RTC_TimeStruct;
    RTC_DateTypeDef RTC_DateStruct;

    // 开启电源接口时钟
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_PWR, ENABLE);
    PWR_BackupAccessCmd(ENABLE);  // 允许访问备份域

    // 开启 LSE
    RCC_LSEConfig(RCC_LSE_ON);
    while (RCC_GetFlagStatus(RCC_FLAG_LSERDY) == RESET);

    // 选择 LSE 为 RTC 时钟源
    RCC_RTCCLKConfig(RCC_RTCCLKSource_LSE);
    RCC_RTCCLKCmd(ENABLE);

    // 等待同步
    RTC_WaitForSynchro();
    RTC_WaitForLastTask();

    // 设置初始时间：2026-08-07 12:00:00
    RTC_TimeStruct.RTC_Hours = 12;
    RTC_TimeStruct.RTC_Minutes = 0;
    RTC_TimeStruct.RTC_Seconds = 0;
    RTC_SetTime(RTC_Format_BIN, &RTC_TimeStruct);

    RTC_DateStruct.RTC_Year = 26;
    RTC_DateStruct.RTC_Month = 8;
    RTC_DateStruct.RTC_Date = 7;
    RTC_DateStruct.RTC_WeekDay = 7;
    RTC_SetDate(RTC_Format_BIN, &RTC_DateStruct);
}
```

### 3.2 读取并显示时间

```c
void RTC_ShowTime(void)
{
    RTC_TimeTypeDef RTC_TimeStruct;
    RTC_DateTypeDef RTC_DateStruct;

    RTC_GetTime(RTC_Format_BIN, &RTC_TimeStruct);
    RTC_GetDate(RTC_Format_BIN, &RTC_DateStruct);

    printf("20%02d-%02d-%02d ", RTC_DateStruct.RTC_Year,
           RTC_DateStruct.RTC_Month, RTC_DateStruct.RTC_Date);
    printf("%02d:%02d:%02d\r\n", RTC_TimeStruct.RTC_Hours,
           RTC_TimeStruct.RTC_Minutes, RTC_TimeStruct.RTC_Seconds);
}
```

---

## 四、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【时钟源】LSE（32.768kHz 晶振）                    │
│  【备份域】需开启 PWR_BackupAccessCmd               │
│  【BCD 格式】注意 BIN 与 BCD 格式转换               │
│  【闹钟】可设置多个闹钟中断                         │
└─────────────────────────────────────────────────────┘
```

---

*上一篇：Day 21 - DAC 模拟输出*  
*下一篇：Day 23 - 看门狗*