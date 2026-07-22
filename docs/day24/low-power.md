# Day 24 - 低功耗模式学习笔记

> 学习日期：2026-08-09

---

## 一、核心知识点

### 1.1 低功耗模式简介

STM32 提供三种低功耗模式，用于降低功耗。

```
┌─────────────────────────────────────────────────────┐
│  模式      │ 功耗  │ 唤醒时间 │ 适用场景           │
├────────────┼───────┼──────────┼────────────────────│
│ 睡眠模式   │ 高    │ 最短     │ 短暂停            │
│ 停止模式   │ 中    │ 中等     │ 长等待            │
│ 待机模式   │ 最低  │ 最长     │ 极低功耗          │
└─────────────────────────────────────────────────────┘
```

### 1.2 模式详解

```
睡眠模式（Sleep）：
    - 仅关闭 CPU 时钟
    - 外设继续运行
    - 任意中断唤醒
    - 功耗：约 10mA

停止模式（Stop）：
    - 关闭所有时钟
    - SRAM 和寄存器保持
    - EXTI 或 RTC 闹钟唤醒
    - 功耗：约 20μA

待机模式（Standby）：
    - 全部断电
    - 仅保留备份域和待机电路
    - WKUP 引脚、RTC 闹钟、NRST 唤醒
    - 功耗：约 2μA
```

---

## 二、常用库函数

```c
// 进入睡眠模式
void PWR_EnterSLEEPMode(uint32_t PWR_Regulator, uint8_t PWR_SLEEPEntry);

// 进入停止模式
void PWR_EnterSTOPMode(uint32_t PWR_Regulator, uint8_t PWR_STOPEntry);

// 进入待机模式
void PWR_EnterStandbyMode(void);

// 使能唤醒引脚
void PWR_WakeUpPinCmd(FunctionalState NewState);

// 清除唤醒标志
void PWR_ClearFlag(uint32_t PWR_FLAG);
```

---

## 三、完整代码实现

### 3.1 睡眠模式

```c
void Enter_Sleep_Mode(void)
{
    // 配置 EXTI 唤醒源
    EXTI_InitTypeDef EXTI_InitStructure;
    EXTI_InitStructure.EXTI_Line = EXTI_Line0;
    EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;
    EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Falling;
    EXTI_InitStructure.EXTI_LineCmd = ENABLE;
    EXTI_Init(&EXTI_InitStructure);

    // 进入睡眠模式
    PWR_EnterSLEEPMode(PWR_Regulator_ON, PWR_SLEEPEntry_WFI);
}

void EXTI0_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line0))
    {
        // 唤醒后处理
        EXTI_ClearITPendingBit(EXTI_Line0);
    }
}
```

### 3.2 停止模式

```c
void Enter_Stop_Mode(void)
{
    // 配置 RTC 闹钟唤醒
    RTC_SetAlarm(RTC_GetCounter() + 10);  // 10 秒后唤醒
    RTC_ITConfig(RTC_IT_ALR, ENABLE);

    // 进入停止模式
    PWR_EnterSTOPMode(PWR_Regulator_LowPower, PWR_STOPEntry_WFI);

    // 唤醒后重新初始化时钟
    SystemInit();
}
```

### 3.3 待机模式

```c
void Enter_Standby_Mode(void)
{
    // 使能 WKUP 引脚唤醒
    PWR_WakeUpPinCmd(ENABLE);

    // 清除唤醒标志
    PWR_ClearFlag(PWR_FLAG_WU);

    // 进入待机模式
    PWR_EnterStandbyMode();

    // 代码不会执行到这里
}
```

---

## 四、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【睡眠模式】仅关闭 CPU，外设继续运行              │
│  【停止模式】SRAM 保持，RTC/EXTI 唤醒              │
│  【待机模式】最低功耗，仅保留备份域                │
│  【唤醒后】停止模式需重新初始化时钟                │
└─────────────────────────────────────────────────────┘
```

---

*上一篇：Day 23 - 看门狗*  
*下一篇：Day 25 - NVIC 中断管理进阶*