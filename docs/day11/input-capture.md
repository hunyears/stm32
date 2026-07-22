# Day 11 - 输入捕获学习笔记

> 学习日期：2026-07-27

---

## 一、核心知识点

### 1.1 什么是输入捕获？

#### 定义

输入捕获（Input Capture）是定时器的一种工作模式，用于**测量外部信号的脉宽或频率**。

```
输入捕获原理：

外部信号 ────────┐
                  │
                  ↓
            ┌───────────┐
            │  定时器    │ ← 计数器持续运行
            │  捕获单元  │
            └───────────┘
                  │
        上升沿/下降沿触发
                  │
                  ↓
            记录当前 CNT 值

用途：
    - 测量 PWM 脉宽
    - 测量信号频率
    - 测量按键按下时长
```

#### 与 PWM 输出的对比

| 特性 | PWM 输出 | 输入捕获 |
|------|----------|----------|
| 方向 | MCU → 外部 | 外部 → MCU |
| 功能 | 产生 PWM 波形 | 测量 PWM 波形 |
| 典型应用 | LED 调光、电机控制 | 遥控器解码、频率测量 |

### 1.2 输入捕获的工作原理

```
测量脉宽的步骤：

        信号 ────┐           ┌────────────
                 │           │
                 └───────────┘
                 ↑           ↑
              上升沿       下降沿
                 │           │
              捕获CNT1    捕获CNT2
                 │           │
                 └─────┬─────┘
                       ↓
              脉宽 = CNT2 - CNT1

时序图：

CNT:  0  100  200  300  400  500  600
      │    │    │    │    │    │    │
信号: └────┐              ┌────────────
           │              │
           ↓              ↓
       捕获CCR1=100   捕获CCR2=500
           │              │
           └──────┬───────┘
                  ↓
           脉宽 = 500 - 100 = 400 个计数周期
```

### 1.3 输入捕获通道

```
STM32F103 定时器输入捕获通道：

TIM2：
    CH1 → PA0
    CH2 → PA1
    CH3 → PA2
    CH4 → PA3

TIM3：
    CH1 → PA6
    CH2 → PA7
    CH3 → PB0
    CH4 → PB1

每个通道可以独立配置为输入捕获模式
```

---

## 二、常用库函数

### 2.1 输入捕获初始化

```c
// 输入捕获初始化
void TIM_ICInit(TIM_TypeDef* TIMx, TIM_ICInitTypeDef* TIM_ICInitStruct);

// 结构体定义
typedef struct {
    uint16_t TIM_Channel;      // 通道选择（TIM_Channel_1~4）
    uint16_t TIM_ICPolarity;   // 触发边沿
    uint16_t TIM_ICSelection;  // 输入选择
    uint16_t TIM_ICPrescaler;  // 预分频
    uint16_t TIM_ICFilter;     // 滤波器
} TIM_ICInitTypeDef;
```

### 2.2 触发边沿配置

```c
// 上升沿触发
TIM_ICPolarity_Rising

// 下降沿触发
TIM_ICPolarity_Falling

// 双边沿触发
TIM_ICPolarity_BothEdge
```

### 2.3 中断相关

```c
// 使能捕获/比较中断
void TIM_ITConfig(TIM_TypeDef* TIMx, uint16_t TIM_IT, FunctionalState NewState);

// 获取中断状态
ITStatus TIM_GetITStatus(TIM_TypeDef* TIMx, uint16_t TIM_IT);

// 清除中断标志
void TIM_ClearITPendingBit(TIM_TypeDef* TIMx, uint16_t TIM_IT);

// 获取捕获值
uint16_t TIM_GetCapture1(TIM_TypeDef* TIMx);  // 通道1
uint16_t TIM_GetCapture2(TIM_TypeDef* TIMx);  // 通道2
```

---

## 三、核心原理

### 3.1 测量脉宽的完整流程

```
步骤1：配置输入捕获
    - 选择通道
    - 设置触发边沿（如上升沿）
    - 开启中断

步骤2：上升沿触发中断
    - 记录 CNT 值到 CCR1
    - 切换为下降沿触发
    - 清零计数器（可选）

步骤3：下降沿触发中断
    - 记录 CNT 值到 CCR2
    - 计算：脉宽 = CCR2 - CCR1

步骤4：处理结果
    - 脉宽时间 = (CCR2 - CCR1) × 计数周期
```

### 3.2 频率测量方法

```
方法一：测量周期法（适合低频）
    测量一个完整周期的时间 T
    频率 = 1 / T

方法二：测量计数法（适合高频）
    在固定时间内统计脉冲个数 N
    频率 = N / 测量时间
```

---

## 四、电路连接

### 4.1 硬件清单

| 组件 | 数量 | 说明 |
|------|------|------|
| STM32F103C8T6 最小系统板 | 1 | - |
| 按键 | 1 | 测量按下时长 |
| 或信号源 | 1 | 如函数发生器 |

### 4.2 推荐接线方案

| STM32 引脚 | 连接 | 说明 |
|------------|------|------|
| PA0 | 按键 → GND | TIM2 CH1 输入捕获 |
| PA6 | 外部信号 | TIM3 CH1 输入捕获 |

### 4.3 电路图

```
        STM32F103C8T6
       ┌─────────────┐
       │             │
       │    PA0 ─────┼────┬──── 按键
       │  (TIM2_CH1) │    │
       │   输入捕获  │  ┌─┴─┐
       │             │  │KEY│
       │             │  └─┬─┘
       │    GND ─────┼────┘
       │             │
       └─────────────┘

功能：测量按键按下的时长
```

---

## 五、完整代码实现

### 5.1 实验一：测量按键按下时长

```c
#include "stm32f10x.h"

volatile uint32_t press_time = 0;    // 按下时长
volatile uint8_t capture_done = 0;   // 捕获完成标志

// ==================== 输入捕获初始化 ====================
void Input_Capture_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    TIM_TimeBaseInitTypeDef TIM_TimeBaseStructure;
    TIM_ICInitTypeDef TIM_ICInitStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

    // 开启时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_AFIO, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);

    // GPIO 配置（上拉输入）
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // 定时器时基配置（1MHz 计数，即 1μs 分辨率）
    TIM_TimeBaseStructure.TIM_Period = 0xFFFF;      // ARR（最大值）
    TIM_TimeBaseStructure.TIM_Prescaler = 71;       // PSC
    TIM_TimeBaseStructure.TIM_ClockDivision = 0;
    TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInit(TIM2, &TIM_TimeBaseStructure);

    // 输入捕获配置（上升沿触发）
    TIM_ICInitStructure.TIM_Channel = TIM_Channel_1;
    TIM_ICInitStructure.TIM_ICPolarity = TIM_ICPolarity_Rising;
    TIM_ICInitStructure.TIM_ICSelection = TIM_ICSelection_DirectTI;
    TIM_ICInitStructure.TIM_ICPrescaler = TIM_ICPSC_DIV1;
    TIM_ICInitStructure.TIM_ICFilter = 0x0;
    TIM_ICInit(TIM2, &TIM_ICInitStructure);

    // 使能捕获中断
    TIM_ITConfig(TIM2, TIM_IT_CC1, ENABLE);

    // NVIC 配置
    NVIC_InitStructure.NVIC_IRQChannel = TIM2_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    // 启动定时器
    TIM_Cmd(TIM2, ENABLE);
}

// ==================== 捕获状态 ====================
typedef enum {
    CAPTURE_STATE_IDLE = 0,    // 空闲
    CAPTURE_STATE_RISING,       // 已捕获上升沿
    CAPTURE_STATE_DONE          // 捕获完成
} Capture_State;

Capture_State capture_state = CAPTURE_STATE_IDLE;
uint16_t rising_value = 0;

// ==================== 中断服务函数 ====================
void TIM2_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM2, TIM_IT_CC1) != RESET)
    {
        TIM_ClearITPendingBit(TIM2, TIM_IT_CC1);

        if (capture_state == CAPTURE_STATE_IDLE)
        {
            // 上升沿（按键按下）
            rising_value = TIM_GetCapture1(TIM2);
            TIM_SetCounter(TIM2, 0);  // 清零计数器

            // 切换为下降沿触发
            TIM2->CCER |= TIM_CCER_CC1P;  // 设置下降沿

            capture_state = CAPTURE_STATE_RISING;
        }
        else if (capture_state == CAPTURE_STATE_RISING)
        {
            // 下降沿（按键释放）
            uint16_t falling_value = TIM_GetCapture1(TIM2);
            press_time = falling_value;  // 单位：微秒

            // 切换回上升沿触发
            TIM2->CCER &= ~TIM_CCER_CC1P;

            capture_state = CAPTURE_STATE_DONE;
            capture_done = 1;
        }
    }
}

// ==================== 主函数 ====================
int main(void)
{
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    Input_Capture_Init();

    while (1)
    {
        if (capture_done)
        {
            capture_done = 0;
            capture_state = CAPTURE_STATE_IDLE;

            // press_time 现在包含按键按下的微秒数
            // 可以通过串口输出或 LED 闪烁显示
        }
    }
}
```

### 5.2 实验二：测量 PWM 频率和占空比

```c
#include "stm32f10x.h"

volatile uint32_t pwm_period = 0;   // PWM 周期（μs）
volatile uint32_t pwm_high = 0;     // 高电平时间（μs）
volatile uint8_t measure_done = 0;

void Input_Capture_Init(void)
{
    // ... 类似上面的配置 ...
}

// 测量 PWM 的状态机
typedef enum {
    MEASURE_IDLE,
    MEASURE_RISING1,  // 第一个上升沿
    MEASURE_FALLING,  // 下降沿
    MEASURE_RISING2,  // 第二个上升沿
} Measure_State;

Measure_State measure_state = MEASURE_IDLE;
uint16_t t_rising1 = 0, t_falling = 0, t_rising2 = 0;

void TIM2_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM2, TIM_IT_CC1) != RESET)
    {
        TIM_ClearITPendingBit(TIM2, TIM_IT_CC1);

        uint16_t current = TIM_GetCapture1(TIM2);

        switch (measure_state)
        {
            case MEASURE_IDLE:
                // 初始化：等待上升沿
                measure_state = MEASURE_RISING1;
                TIM_SetCounter(TIM2, 0);
                break;

            case MEASURE_RISING1:
                t_rising1 = 0;  // 第一个上升沿作为参考点
                // 切换到下降沿
                TIM2->CCER |= TIM_CCER_CC1P;
                measure_state = MEASURE_FALLING;
                break;

            case MEASURE_FALLING:
                t_falling = current;  // 下降沿时刻
                // 切换到上升沿
                TIM2->CCER &= ~TIM_CCER_CC1P;
                measure_state = MEASURE_RISING2;
                break;

            case MEASURE_RISING2:
                t_rising2 = current;  // 第二个上升沿时刻
                pwm_period = t_rising2;  // 周期
                pwm_high = t_falling;    // 高电平时间
                measure_done = 1;
                measure_state = MEASURE_IDLE;
                break;
        }
    }
}

int main(void)
{
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
    Input_Capture_Init();

    while (1)
    {
        if (measure_done)
        {
            measure_done = 0;

            // 计算频率和占空比
            uint32_t frequency = 1000000 / pwm_period;  // Hz
            float duty = (float)pwm_high / pwm_period * 100;  // %

            // 输出结果（需要串口）
        }
    }
}
```

---

## 六、调试技巧

### 6.1 常见问题排查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| 无法捕获 | GPIO 模式错误 | 使用输入模式 |
| 捕获值不对 | 触发边沿配置错误 | 检查 CCER 寄存器 |
| 溢出问题 | 测量时间过长 | 使用定时器溢出中断 |
| 分辨率不够 | 预分频太大 | 减小 PSC 值 |

### 6.2 溢出处理

```c
// 处理计数器溢出
volatile uint32_t overflow_count = 0;

void TIM2_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM2, TIM_IT_Update) != RESET)
    {
        overflow_count++;
        TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
    }

    if (TIM_GetITStatus(TIM2, TIM_IT_CC1) != RESET)
    {
        // 捕获处理，考虑溢出
        uint32_t total = overflow_count * 0x10000 + TIM_GetCapture1(TIM2);
        // ...
    }
}
```

---

## 七、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【输入捕获用途】测量脉宽、频率                     │
│  【工作原理】边沿触发时记录 CNT 值                  │
│  【触发边沿】上升沿、下降沿、双边沿                 │
│  【计算方法】脉宽 = CCR2 - CCR1                    │
│  【溢出处理】长时间测量需要处理计数器溢出           │
└─────────────────────────────────────────────────────┘
```

### 记忆口诀

```
输入捕获测脉宽，边沿触发记时间。
上升沿来记起点，下降沿来算终点。
溢出处理要记牢，频率测量用它好。
```

---

## 八、拓展练习

1. **遥控器解码**：使用输入捕获解码红外遥控器信号
2. **频率计**：制作一个简易频率计
3. **电机转速测量**：配合霍尔传感器测量电机转速
4. **超声波测距**：测量超声波回波脉宽计算距离

---

*上一篇：Day 10 - PWM 舵机控制*  
*下一篇：Day 12 - USART 串口基础*