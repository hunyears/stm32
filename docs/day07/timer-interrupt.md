# Day 07 - 定时器中断学习笔记

> 学习日期：2026-07-23

---

## 一、核心知识点

### 1.1 什么是定时器？

#### 类比理解

```
定时器就像一个"精准的秒表"：

【软件延时（Day 02）】
你用数数的方式计时："一秒、两秒..."
问题：不准确，而且数数时不能做其他事

【硬件定时器】
你有一个电子秒表，设定好时间后会自动响铃
期间你可以做其他事，响铃时再处理
```

#### STM32 定时器分类

| 类型 | 定时器 | 数量 | 特点 |
|------|--------|------|------|
| **基本定时器** | TIM6, TIM7 | 2 | 最简单，只能定时 |
| **通用定时器** | TIM2~TIM5 | 4 | 定时 + PWM + 输入捕获 |
| **高级定时器** | TIM1, TIM8 | 2 | 通用功能 + 死区控制 + 刹车 |

### 1.2 定时器的工作原理

#### 计数器原理

```
                    时钟源（如 72MHz）
                          │
                          ↓
                    ┌─────────┐
                    │ 预分频器 │ ← 分频系数（PSC）
                    └────┬────┘
                         │
                         ↓  计数频率 = 时钟源 / (PSC + 1)
                    ┌─────────┐
                    │ 计数器   │ ← 计数值（CNT）
                    └────┬────┘
                         │
                         ↓  计数到自动重装载值时产生中断
                    ┌─────────┐
                    │自动重装载│ ← 重装载值（ARR）
                    └─────────┘
                         │
                         ↓
                    产生中断/事件

定时周期计算：
    定时时间 = (PSC + 1) × (ARR + 1) / 时钟源频率

示例（72MHz 时钟，定时 1 秒）：
    PSC = 7199   → 分频后 10kHz
    ARR = 9999   → 计数 10000 次
    定时时间 = 7200 × 10000 / 72000000 = 1 秒
```

### 1.3 定时器中断类型

| 中断类型 | 宏定义 | 说明 |
|----------|--------|------|
| 更新中断 | `TIM_IT_Update` | 计数器溢出/重装载 |
| 捕获/比较中断 | `TIM_IT_CC1~4` | 输入捕获/输出比较 |
| 触发中断 | `TIM_IT_Trigger` | 外部触发事件 |

---

## 二、常用库函数

### 2.1 定时器时基配置

```c
// 开启定时器时钟（APB1 总线）
RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);

// 定时器时基初始化
void TIM_TimeBaseInit(TIM_TypeDef* TIMx, TIM_TimeBaseInitTypeDef* TIM_TimeBaseInitStruct);

// 结构体定义
typedef struct {
    uint16_t TIM_Prescaler;         // 预分频值（0-65535）
    uint16_t TIM_CounterMode;       // 计数模式（向上/向下/中央对齐）
    uint16_t TIM_Period;            // 自动重装载值（0-65535）
    uint16_t TIM_ClockDivision;     // 时钟分频（用于滤波）
    uint8_t TIM_RepetitionCounter;  // 重复计数器（仅高级定时器）
} TIM_TimeBaseInitTypeDef;
```

### 2.2 中断配置

```c
// 配置定时器中断
void TIM_ITConfig(TIM_TypeDef* TIMx, uint16_t TIM_IT, FunctionalState NewState);

// 获取中断标志状态
ITStatus TIM_GetITStatus(TIM_TypeDef* TIMx, uint16_t TIM_IT);

// 清除中断标志
void TIM_ClearITPendingBit(TIM_TypeDef* TIMx, uint16_t TIM_IT);

// 使能/禁用定时器
void TIM_Cmd(TIM_TypeDef* TIMx, FunctionalState NewState);
```

### 2.3 计数模式

```c
// 向上计数（默认）：0 → ARR → 中断 → 0
TIM_CounterMode_Up

// 向下计数：ARR → 0 → 中断 → ARR
TIM_CounterMode_Down

// 中央对齐：0 → ARR → 0，中间有两次比较事件
TIM_CounterMode_CenterAligned1/2/3
```

---

## 三、核心原理

### 3.1 定时周期计算

```
公式：
    T = (PSC + 1) × (ARR + 1) / Fclk

    T     - 定时周期（秒）
    PSC   - 预分频值
    ARR   - 自动重装载值
    Fclk  - 时钟源频率（Hz）

常用配置速查表（72MHz 时钟）：

┌────────────────────────────────────────────────────────┐
│  定时时间  │   PSC    │    ARR    │      说明         │
├────────────┼──────────┼───────────┼───────────────────│
│  1 ms      │   71     │   999     │  分频 72，计数 1000│
│  10 ms     │  7199    │   99      │  分频 7200，计数 100│
│  100 ms    │  7199    │   999     │  分频 7200，计数 1000│
│  1 s       │  7199    │   9999    │  分频 7200，计数 10000│
│  1 s       │  71999   │   999     │  分频 72000，计数 1000│
└────────────────────────────────────────────────────────┘

计算示例（定时 1ms）：
    目标：定时 1ms = 0.001s
    
    方案1：PSC = 71, ARR = 999
    计数频率 = 72MHz / 72 = 1MHz（每 1μs 计数一次）
    定时时间 = 1000 × 1μs = 1000μs = 1ms
    
    方案2：PSC = 7199, ARR = 9
    计数频率 = 72MHz / 7200 = 10kHz（每 100μs 计数一次）
    定时时间 = 10 × 100μs = 1000μs = 1ms
```

### 3.2 多种定时需求的处理

```
问题：需要多个不同的定时时间，怎么办？

方案1：多个定时器
    TIM2 定时 1ms
    TIM3 定时 10ms
    TIM4 定时 100ms
    优点：简单直接
    缺点：占用定时器资源

方案2：一个定时器 + 软件计数
    TIM2 定时 1ms
    中断中：
        count1++ → 到 10 就是 10ms
        count2++ → 到 100 就是 100ms
        count3++ → 到 1000 就是 1s
    优点：节省硬件资源
    缺点：需要软件管理
```

---

## 四、电路连接

### 4.1 硬件清单

| 组件 | 数量 | 说明 |
|------|------|------|
| STM32F103C8T6 最小系统板 | 1 | 已板载 LED（PC13） |
| ST-Link | 1 | 程序下载与调试 |

### 4.2 电路图

```
        STM32F103C8T6
       ┌─────────────┐
       │             │
       │    PC13 ────┼────●─── 板载 LED
       │             │
       │   内部定时器│
       │   TIM2      │
       │             │
       └─────────────┘

说明：本实验使用内部定时器，无需外部电路
```

---

## 五、完整代码实现

### 5.1 实验一：1秒定时翻转 LED

```c
#include "stm32f10x.h"

// ==================== 定时器初始化 ====================
void TIM2_Config(void)
{
    TIM_TimeBaseInitTypeDef TIM_TimeBaseStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

    // 1. 开启 TIM2 时钟（APB1 总线）
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);

    // 2. 时基配置：定时 1 秒
    //    72MHz / 7200 = 10kHz，计数 10000 次 = 1 秒
    TIM_TimeBaseStructure.TIM_Period = 9999;           // ARR
    TIM_TimeBaseStructure.TIM_Prescaler = 7199;        // PSC
    TIM_TimeBaseStructure.TIM_ClockDivision = 0;
    TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInit(TIM2, &TIM_TimeBaseStructure);

    // 3. 使能更新中断
    TIM_ITConfig(TIM2, TIM_IT_Update, ENABLE);

    // 4. 配置 NVIC
    NVIC_InitStructure.NVIC_IRQChannel = TIM2_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    // 5. 使能定时器
    TIM_Cmd(TIM2, ENABLE);
}

// ==================== GPIO 初始化 ====================
void GPIO_Config(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOC, &GPIO_InitStructure);
}

// ==================== 主函数 ====================
int main(void)
{
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);

    GPIO_Config();
    TIM2_Config();

    while (1)
    {
        // 主循环空闲，可以执行其他任务
    }
}

// ==================== 定时器中断服务函数 ====================
void TIM2_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM2, TIM_IT_Update) != RESET)
    {
        // 翻转 LED
        GPIO_WriteBit(GPIOC, GPIO_Pin_13,
            (BitAction)(1 - GPIO_ReadOutputDataBit(GPIOC, GPIO_Pin_13)));

        // 清除中断标志
        TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
    }
}
```

### 5.2 实验二：多定时任务（软件计数）

```c
#include "stm32f10x.h"

// 定时任务标志
volatile uint8_t flag_10ms = 0;
volatile uint8_t flag_100ms = 0;
volatile uint8_t flag_1s = 0;

void TIM2_Config(void)
{
    TIM_TimeBaseInitTypeDef TIM_TimeBaseStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);

    // 定时 1ms（基准）
    TIM_TimeBaseStructure.TIM_Period = 9;      // ARR
    TIM_TimeBaseStructure.TIM_Prescaler = 7199; // PSC
    TIM_TimeBaseStructure.TIM_ClockDivision = 0;
    TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInit(TIM2, &TIM_TimeBaseStructure);

    TIM_ITConfig(TIM2, TIM_IT_Update, ENABLE);

    NVIC_InitStructure.NVIC_IRQChannel = TIM2_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    TIM_Cmd(TIM2, ENABLE);
}

// 软件计数器
void TIM2_IRQHandler(void)
{
    static uint16_t count = 0;

    if (TIM_GetITStatus(TIM2, TIM_IT_Update) != RESET)
    {
        count++;

        if (count % 10 == 0)   flag_10ms = 1;   // 每 10ms
        if (count % 100 == 0)  flag_100ms = 1;  // 每 100ms
        if (count % 1000 == 0) flag_1s = 1;     // 每 1s

        TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
    }
}

int main(void)
{
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    // ... GPIO 初始化 ...

    while (1)
    {
        if (flag_10ms)
        {
            flag_10ms = 0;
            // 执行 10ms 任务（如按键扫描）
        }

        if (flag_100ms)
        {
            flag_100ms = 0;
            // 执行 100ms 任务
        }

        if (flag_1s)
        {
            flag_1s = 0;
            // 执行 1s 任务（如 LED 翻转）
        }
    }
}
```

### 5.3 实验三：精确延时函数（非阻塞）

```c
#include "stm32f10x.h"

// 非阻塞延时结构
typedef struct {
    uint32_t start;
    uint32_t interval;
    uint8_t active;
} Delay_TypeDef;

// 获取系统 ticks（假设 1ms 定时器中断已开启）
volatile uint32_t system_ticks = 0;

void TIM2_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM2, TIM_IT_Update) != RESET)
    {
        system_ticks++;
        TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
    }
}

// 启动非阻塞延时
void Delay_Start(Delay_TypeDef* delay, uint32_t ms)
{
    delay->start = system_ticks;
    delay->interval = ms;
    delay->active = 1;
}

// 检查延时是否到期
uint8_t Delay_Check(Delay_TypeDef* delay)
{
    if (!delay->active) return 0;

    if ((system_ticks - delay->start) >= delay->interval)
    {
        delay->active = 0;
        return 1;
    }
    return 0;
}

// 使用示例
int main(void)
{
    Delay_TypeDef led_delay;

    // 初始化...

    Delay_Start(&led_delay, 500);  // 500ms 延时

    while (1)
    {
        if (Delay_Check(&led_delay))
        {
            GPIO_WriteBit(GPIOC, GPIO_Pin_13,
                (BitAction)(1 - GPIO_ReadOutputDataBit(GPIOC, GPIO_Pin_13)));
            Delay_Start(&led_delay, 500);  // 重新开始
        }

        // 可以同时做其他事
    }
}
```

---

## 六、调试技巧

### 6.1 常见问题排查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| 定时不准确 | PSC/ARR 计算错误 | 检查时钟频率和分频系数 |
| 中断不触发 | NVIC 未配置 | 检查 NVIC_Init 是否调用 |
| 定时器不工作 | 时钟未开启 | 检查 RCC_APB1PeriphClockCmd |
| 中断频率过快 | ARR 值太小 | 增大 ARR 值 |

### 6.2 验证定时精度

```c
// 使用逻辑分析仪或示波器测量 LED 翻转频率
// 理论值：0.5Hz（1秒亮 + 1秒灭 = 2秒周期）
```

---

## 七、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【定时公式】T = (PSC+1) × (ARR+1) / Fclk          │
│  【时钟总线】TIM2-7 在 APB1，TIM1/8 在 APB2        │
│  【计数方向】向上计数：0 → ARR → 中断              │
│  【中断类型】TIM_IT_Update 更新中断最常用          │
│  【清除标志】必须调用 TIM_ClearITPendingBit        │
└─────────────────────────────────────────────────────┘
```

### 初始化流程图

```
    开始
      │
      ↓
┌─────────────────────┐
│ 开启定时器时钟      │  ← RCC_APB1PeriphClockCmd
│ （APB1 总线）       │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ TIM_TimeBaseInit    │  ← 配置 PSC、ARR、计数模式
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ TIM_ITConfig        │  ← 使能更新中断
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ NVIC_Init           │  ← 配置中断通道、优先级
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ TIM_Cmd             │  ← 启动定时器
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ 等待中断            │
└─────────────────────┘
```

### 记忆口诀

```
定时器就是计数器，分频之后数个数。
数到ARR就中断，时间计算要记牢。
PSC加一乘ARR加一，除以时钟得时间。
```

---

## 八、拓展练习

1. **精确秒表**：使用定时器实现一个精确的秒表，精确到毫秒
2. **可调频率闪烁**：通过按键调节 LED 闪烁频率（改变 ARR 值）
3. **多任务调度器**：基于定时器实现一个简单的多任务调度框架
4. ** PWM 预备**：理解定时器的输出比较模式（为 Day 08 做准备）

---

*上一篇：Day 06 - 外部中断 EXTI*  
*下一篇：Day 08 - PWM 输出基础*