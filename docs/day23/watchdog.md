# Day 23 - 看门狗学习笔记

> 学习日期：2026-08-08

---

## 一、核心知识点

### 1.1 什么是看门狗？

看门狗（Watchdog）是一种**系统监控机制**，用于检测和恢复程序跑飞或死循环。

```
工作原理：

    ┌─────────────────────────────────────────────┐
    │  主程序正常运行                             │
    │       │                                    │
    │       ↓                                    │
    │  喂狗（重置计数器）                         │
    │       │                                    │
    │       ↓                                    │
    │  继续执行...                               │
    │       │                                    │
    │       ↓                                    │
    │  如果程序卡死，无法喂狗                     │
    │       │                                    │
    │       ↓                                    │
    │  计数器溢出 → 复位系统                     │
    └─────────────────────────────────────────────┘

比喻：
    看门狗 = 计时炸弹
    喂狗 = 重置计时器
    如果不及时喂狗 → 炸弹爆炸（系统复位）
```

### 1.2 两种看门狗

| 特性 | 独立看门狗（IWDG） | 窗口看门狗（WWDG） |
|------|-------------------|-------------------|
| 时钟源 | LSI（内部低速） | APB1（系统时钟） |
| 复位条件 | 计数器到 0 | 计数器到 0 或提前喂狗 |
| 适用场景 | 独立监控、安全性高 | 精确时间监控 |
| 低功耗 | 可用 | 不可用 |

---

## 二、常用库函数

### 2.1 独立看门狗

```c
// 写入使能
void IWDG_WriteAccessCmd(uint16_t IWDG_WriteAccess);

// 设置预分频
void IWDG_SetPrescaler(uint8_t IWDG_Prescaler);

// 设置重装载值
void IWDG_SetReload(uint16_t Reload);

// 重载计数器（喂狗）
void IWDG_ReloadCounter(void);

// 使能看门狗
void IWDG_Enable(void);
```

### 2.2 窗口看门狗

```c
// 初始化
void WWDG_Init(uint8_t Prescaler, uint8_t Window, uint8_t Counter);

// 使能
void WWDG_Enable(uint8_t Counter);

// 喂狗
void WWDG_SetCounter(uint8_t Counter);

// 中断配置
void WWDG_EnableIT(void);
```

---

## 三、完整代码实现

### 3.1 独立看门狗

```c
void IWDG_Init(uint8_t prescaler, uint16_t reload)
{
    IWDG_WriteAccessCmd(IWDG_WriteAccess_Enable);
    IWDG_SetPrescaler(prescaler);
    IWDG_SetReload(reload);
    IWDG_ReloadCounter();
    IWDG_Enable();
}

// 喂狗
void IWDG_Feed(void)
{
    IWDG_ReloadCounter();
}

int main(void)
{
    IWDG_Init(IWDG_Prescaler_256, 1000);  // 约 4 秒

    while (1)
    {
        // 正常任务
        do_something();

        // 喂狗
        IWDG_Feed();
    }
}
```

### 3.2 窗口看门狗

```c
void WWDG_Init_Config(void)
{
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_WWDG, ENABLE);

    WWDG_SetPrescaler(WWDG_Prescaler_8);
    WWDG_SetWindowValue(80);
    WWDG_Enable(127);

    // 使能中断
    NVIC_EnableIRQ(WWDG_IRQn);
    WWDG_EnableIT();
}

void WWDG_IRQHandler(void)
{
    // 紧急喂狗或处理
    WWDG_SetCounter(127);
    WWDG_ClearFlag();
}
```

---

## 四、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【IWDG 时钟】LSI 内部低速时钟（约 40kHz）          │
│  【IWDG 复位】计数器到 0 时复位                     │
│  【WWDG 窗口】必须在窗口时间内喂狗                  │
│  【喂狗时机】关键任务完成后喂狗                     │
└─────────────────────────────────────────────────────┘
```

---

*上一篇：Day 22 - RTC 实时时钟*  
*下一篇：Day 24 - 低功耗模式*