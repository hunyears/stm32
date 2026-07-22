# Day 09 - PWM 呼吸灯学习笔记

> 学习日期：2026-07-25

---

## 一、核心知识点

### 1.1 什么是呼吸灯？

#### 效果描述

```
呼吸灯效果：LED 亮度逐渐增强（吸气），然后逐渐减弱（呼气），循环往复。

亮度变化曲线：

亮度
  100%│    ╱╲      ╱╲      ╱╲
      │   ╱  ╲    ╱  ╲    ╱  ╲
   50%│  ╱    ╲  ╱    ╲  ╱    ╲
      │ ╱      ╲╱      ╲╱      ╲
    0%└──────────────────────────→ 时间
        吸气  呼气  吸气  呼气

实现方式：PWM 占空比从 0% 渐变到 100%，再从 100% 渐变到 0%
```

### 1.2 呼吸灯的实现方式

#### 方式一：延时循环（简单但不精确）

```c
while (1) {
    // 亮度渐增
    for (int i = 0; i <= 100; i++) {
        PWM_SetDuty(i);
        delay_ms(10);
    }
    // 亮度渐减
    for (int i = 100; i >= 0; i--) {
        PWM_SetDuty(i);
        delay_ms(10);
    }
}
```

#### 方式二：定时器中断（精确且非阻塞）

```c
// 在定时器中断中更新占空比
void TIM_IRQHandler(void) {
    static uint8_t brightness = 0;
    static int8_t direction = 1;
    
    brightness += direction;
    if (brightness >= 100 || brightness <= 0) {
        direction = -direction;
    }
    
    PWM_SetDuty(brightness);
}
```

### 1.3 人眼视觉暂留

```
问题：为什么 100Hz 的 PWM 就能实现平滑调光？

答案：人眼的视觉暂留效应

人眼特点：
    - 刷新率约 24Hz 以上就被视为连续
    - PWM 频率越高，闪烁感越弱
    - LED 调光常用 100Hz~1kHz

视觉暂留原理：
    ┌────────────────────────────────────────┐
    │  人眼看到的不是瞬时亮度，而是"平均亮度"│
    │                                        │
    │  占空比 50% = 看到的亮度约为 50%       │
    │  占空比 25% = 看到的亮度约为 25%       │
    └────────────────────────────────────────┘

呼吸灯渐变步长选择：
    - 步长太小：变化太慢，一个周期很长
    - 步长太大：变化不平滑，有跳变感
    - 推荐：每 10-20ms 变化 1%
```

---

## 二、常用库函数

### 2.1 PWM 相关函数（复习）

```c
// 设置比较值（改变占空比）
void TIM_SetCompare1(TIM_TypeDef* TIMx, uint16_t Compare1);

// 使能定时器
void TIM_Cmd(TIM_TypeDef* TIMx, FunctionalState NewState);

// 定时器中断配置
void TIM_ITConfig(TIM_TypeDef* TIMx, uint16_t TIM_IT, FunctionalState NewState);
```

### 2.2 延时函数

```c
// 简易延时（阻塞式）
void delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 8000; j++);
}
```

---

## 三、核心原理

### 3.1 亮度变化数学模型

```
线性渐变：
    brightness(t) = brightness(t-1) + step

非线性渐变（更自然）：
    人眼对亮度感知是非线性的
    可以使用 Gamma 校正

Gamma 校正公式：
    output = input ^ gamma
    gamma ≈ 2.2（人眼特性）

简化实现：
    // 查表法（预计算好的 Gamma 校正表）
    const uint16_t gamma_table[101] = {
        0, 0, 0, 0, 1, 1, 1, 1, 2, 2,
        // ... 预计算 101 个值
    };
    
    实际 CCR = gamma_table[brightness];
```

### 3.2 呼吸周期计算

```
一个完整呼吸周期 = 亮度从 0→100→0

参数计算：
    - 占空比范围：0% ~ 100%（对应 CCR：0 ~ ARR）
    - 步长：每 10ms 变化 1%
    - 单向时间：100 步 × 10ms = 1 秒
    - 完整周期：2 秒（吸气 1 秒 + 呼气 1 秒）

调整方法：
    1. 改变步长 → 改变呼吸速度
    2. 改变更新间隔 → 改变平滑度
```

---

## 四、电路连接

### 4.1 硬件清单

| 组件 | 数量 | 说明 |
|------|------|------|
| STM32F103C8T6 最小系统板 | 1 | - |
| LED | 1 | 外接或使用板载 |
| 杜邦线 | 若干 | 连接电路 |

### 4.2 推荐接线方案

| STM32 引脚 | 连接 | 说明 |
|------------|------|------|
| PA0 | LED → GND | TIM2 CH1 PWM 输出 |
| 或 PC13 | 板载 LED | 低电平亮，需反转 |

### 4.3 电路图

```
        STM32F103C8T6
       ┌─────────────┐
       │             │
       │    PA0 ─────┼────┬──── LED
       │  (TIM2_CH1) │    │
       │             │    R
       │             │    │
       │    GND ─────┼────┴──── GND
       │             │
       └─────────────┘
```

---

## 五、完整代码实现

### 5.1 实验一：基本呼吸灯（延时方式）

```c
#include "stm32f10x.h"

// ==================== PWM 初始化 ====================
void PWM_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    TIM_TimeBaseInitTypeDef TIM_TimeBaseStructure;
    TIM_OCInitTypeDef TIM_OCInitStructure;

    // 开启时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);

    // GPIO 配置
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // 时基配置（1kHz PWM）
    TIM_TimeBaseStructure.TIM_Period = 999;
    TIM_TimeBaseStructure.TIM_Prescaler = 71;
    TIM_TimeBaseStructure.TIM_ClockDivision = 0;
    TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInit(TIM2, &TIM_TimeBaseStructure);

    // PWM 配置
    TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;
    TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;
    TIM_OCInitStructure.TIM_Pulse = 0;
    TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_High;
    TIM_OC1Init(TIM2, &TIM_OCInitStructure);

    TIM_OC1PreloadConfig(TIM2, TIM_OCPreload_Enable);
    TIM_ARRPreloadConfig(TIM2, ENABLE);
    TIM_Cmd(TIM2, ENABLE);
}

// ==================== 延时函数 ====================
void delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 8000; j++);
}

// ==================== 设置占空比 ====================
void PWM_SetDuty(uint16_t duty)
{
    // duty: 0-100，对应 CCR: 0-999
    uint16_t ccr = duty * 10;
    if (ccr > 999) ccr = 999;
    TIM_SetCompare1(TIM2, ccr);
}

// ==================== 主函数 ====================
int main(void)
{
    int16_t brightness = 0;
    int8_t direction = 1;

    PWM_Init();

    while (1)
    {
        // 更新亮度
        brightness += direction;
        
        // 到达边界时反向
        if (brightness >= 100)
        {
            brightness = 100;
            direction = -1;
        }
        else if (brightness <= 0)
        {
            brightness = 0;
            direction = 1;
        }

        // 设置 PWM 占空比
        PWM_SetDuty(brightness);

        // 延时 10ms（控制呼吸速度）
        delay_ms(10);
    }
}
```

### 5.2 实验二：定时器中断驱动呼吸灯

```c
#include "stm32f10x.h"

volatile uint8_t brightness = 0;
volatile int8_t direction = 1;

// ==================== PWM 初始化 ====================
void PWM_Init(void)
{
    // ... 同上 ...
}

// ==================== 定时器中断初始化 ====================
void TIM3_Config(void)
{
    TIM_TimeBaseInitTypeDef TIM_TimeBaseStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

    // 开启 TIM3 时钟
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM3, ENABLE);

    // 配置 10ms 定时
    TIM_TimeBaseStructure.TIM_Period = 99;      // ARR
    TIM_TimeBaseStructure.TIM_Prescaler = 7199; // PSC（10kHz 计数）
    TIM_TimeBaseStructure.TIM_ClockDivision = 0;
    TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInit(TIM3, &TIM_TimeBaseStructure);

    // 使能更新中断
    TIM_ITConfig(TIM3, TIM_IT_Update, ENABLE);

    // 配置 NVIC
    NVIC_InitStructure.NVIC_IRQChannel = TIM3_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    // 启动定时器
    TIM_Cmd(TIM3, ENABLE);
}

// ==================== 定时器中断服务函数 ====================
void TIM3_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM3, TIM_IT_Update) != RESET)
    {
        // 更新亮度
        brightness += direction;

        if (brightness >= 100)
        {
            brightness = 100;
            direction = -1;
        }
        else if (brightness <= 0)
        {
            brightness = 0;
            direction = 1;
        }

        // 更新 PWM
        TIM_SetCompare1(TIM2, brightness * 10);

        // 清除中断标志
        TIM_ClearITPendingBit(TIM3, TIM_IT_Update);
    }
}

// ==================== 主函数 ====================
int main(void)
{
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);

    PWM_Init();
    TIM3_Config();

    while (1)
    {
        // 主循环可以执行其他任务
        // 呼吸灯完全由中断驱动
    }
}
```

### 5.3 实验三：Gamma 校正呼吸灯

```c
#include "stm32f10x.h"

// Gamma 校正查找表（101 个值，对应 0-100%）
const uint16_t gamma_table[101] = {
    0,   0,   0,   0,   1,   1,   1,   2,   2,   3,
    3,   4,   5,   6,   7,   8,   9,   11,  12,  14,
    15,  17,  19,  21,  23,  25,  28,  30,  33,  35,
    38,  41,  44,  47,  50,  54,  57,  61,  65,  68,
    72,  76,  80,  85,  89,  94,  98,  103, 108, 113,
    118, 123, 129, 134, 140, 146, 152, 158, 164, 170,
    176, 183, 189, 196, 203, 209, 216, 224, 231, 238,
    246, 253, 261, 269, 277, 285, 293, 301, 309, 318,
    326, 335, 344, 353, 362, 371, 380, 389, 399, 408,
    418, 428, 438, 448, 458, 469, 479, 490, 501, 512,
    999
};

void PWM_SetDuty_Gamma(uint8_t brightness)
{
    if (brightness > 100) brightness = 100;
    TIM_SetCompare1(TIM2, gamma_table[brightness]);
}

int main(void)
{
    int16_t brightness = 0;
    int8_t direction = 1;

    PWM_Init();

    while (1)
    {
        brightness += direction;

        if (brightness >= 100)
        {
            brightness = 100;
            direction = -1;
        }
        else if (brightness <= 0)
        {
            brightness = 0;
            direction = 1;
        }

        PWM_SetDuty_Gamma(brightness);
        delay_ms(10);
    }
}
```

---

## 六、调试技巧

### 6.1 常见问题排查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| 亮度变化不平滑 | 步长太大 | 减小步长或增加更新频率 |
| 呼吸速度不对 | 延时时间错误 | 调整 delay_ms 参数 |
| 低亮度时闪烁 | PWM 频率太低 | 提高 PWM 频率到 1kHz 以上 |
| 亮度看起来不均匀 | 未做 Gamma 校正 | 使用 Gamma 校正表 |

### 6.2 参数调优

```
调整呼吸速度：
    - 增大延时 → 呼吸变慢
    - 减小延时 → 呼吸变快
    
调整平滑度：
    - 减小步长 + 增加更新频率 → 更平滑
    - 增大步长 + 减少更新频率 → 更跳跃

推荐配置：
    - 更新间隔：10-20ms
    - 步长：1%（每次更新变化 1%）
    - 呼吸周期：约 2 秒
```

---

## 七、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【呼吸灯本质】占空比连续渐变                        │
│  【实现方式】延时循环或定时器中断                    │
│  【视觉暂留】人眼看到的是平均亮度                    │
│  【Gamma 校正】让亮度变化更自然                      │
│  【参数调整】延时时间控制呼吸速度                    │
└─────────────────────────────────────────────────────┘
```

### 两种实现方式对比

```
┌─────────────────────────────────────────────────────┐
│  方式        │  延时循环  │  定时器中断             │
├──────────────┼────────────┼─────────────────────────│
│  优点        │  简单直接  │  精确、非阻塞           │
│  缺点        │  阻塞      │  稍复杂                 │
│  CPU 占用    │  高        │  低                     │
│  适用场景    │  单任务    │  多任务                 │
└─────────────────────────────────────────────────────┘
```

### 记忆口诀

```
呼吸灯就是占空比变，从小到大再从小。
延时决定速度快，Gamma校正更自然。
定时器做更精确，主程序还能做其他。
```

---

## 八、拓展练习

1. **多色呼吸灯**：RGB LED 实现彩色呼吸效果
2. **可调速度呼吸灯**：按键调节呼吸速度
3. **非线性呼吸**：实现加速-减速的呼吸效果
4. **双 LED 呼吸**：两个 LED 交替呼吸（此消彼长）

---

*上一篇：Day 08 - PWM 输出基础*  
*下一篇：Day 10 - PWM 舵机控制*