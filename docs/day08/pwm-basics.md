# Day 08 - PWM 输出基础学习笔记

> 学习日期：2026-07-24

---

## 一、核心知识点

### 1.1 什么是 PWM？

#### 定义

**PWM（Pulse Width Modulation）**：脉冲宽度调制，一种通过改变脉冲宽度来控制模拟量的技术。

```
PWM 波形：

        ┌──┐    ┌──┐    ┌──┐
        │  │    │  │    │  │
────┘  └────┘  └────┘  └────
    │← T →│
    
T = 周期（固定）
高电平时间 = 脉冲宽度（可变）
```

#### 占空比

```
占空比 = 高电平时间 / 周期 × 100%

示例：
    ┌──────┐        
    │      │        
────┘      └───────
    │←  T  →│

占空比 = 60%：高电平占周期的 60%

占空比决定"平均输出电压"：
    - 0%   → 0V
    - 50%  → 1.65V（3.3V 的一半）
    - 100% → 3.3V
```

### 1.2 PWM 的应用场景

| 应用 | 说明 | 典型频率 |
|------|------|----------|
| LED 调光 | 控制亮度 | 1kHz - 10kHz |
| 电机调速 | 控制转速 | 10kHz - 20kHz |
| 舵机控制 | 控制角度 | 50Hz |
| 音频播放 | 产生声音 | 20Hz - 20kHz |
| DAC 输出 | 模拟电压输出 | 可变 |

### 1.3 STM32 的 PWM 输出

#### 定时器与 PWM 通道

```
STM32F103 通用定时器 PWM 通道：

TIM2：CH1(PA0)  CH2(PA1)  CH3(PA2)  CH4(PA3)
TIM3：CH1(PA6)  CH2(PA7)  CH3(PB0)  CH4(PB1)
TIM4：CH1(PB6)  CH2(PB7)  CH3(PB8)  CH4(PB9)

每个定时器有 4 个通道，可以独立输出 PWM
```

#### PWM 输出原理

```
                    时钟源（如 72MHz）
                          │
                          ↓
                    ┌─────────┐
                    │ 预分频器 │ ← PSC
                    └────┬────┘
                         │
                         ↓
                    ┌─────────┐
                    │ 计数器   │ ← CNT（不断计数）
                    └────┬────┘
                         │
              ┌──────────┼──────────┐
              ↓          ↓          ↓
         ┌────────┐ ┌────────┐ ┌────────┐
         │  CCR1  │ │  CCR2  │ │  CCR3  │ ← 比较寄存器
         └────┬───┘ └────┬───┘ └────┬───┘
              │          │          │
              ↓          ↓          ↓
         ┌────────┐ ┌────────┐ ┌────────┐
         │ 比较器 │ │ 比较器 │ │ 比较器 │
         └────┬───┘ └────┬───┘ └────┬───┘
              │          │          │
              ↓          ↓          ↓
            PWM1       PWM2       PWM3

工作原理：
    当 CNT < CCRx 时：输出一种电平
    当 CNT ≥ CCRx 时：输出另一种电平
    
    CCRx 的值决定占空比
    ARR 的值决定周期
```

---

## 二、常用库函数

### 2.1 PWM 初始化相关

```c
// 定时器时基初始化
void TIM_TimeBaseInit(TIM_TypeDef* TIMx, TIM_TimeBaseInitTypeDef* TIM_TimeBaseInitStruct);

// 输出比较初始化（PWM 模式）
void TIM_OCInit(TIM_TypeDef* TIMx, TIM_OCInitTypeDef* TIM_OCInitStruct);

// 使能预装载寄存器
void TIM_OC1PreloadConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPreload);
void TIM_OC2PreloadConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPreload);
void TIM_OC3PreloadConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPreload);
void TIM_OC4PreloadConfig(TIM_TypeDef* TIMx, uint16_t TIM_OCPreload);

// 使能自动重装载预装载
void TIM_ARRPreloadConfig(TIM_TypeDef* TIMx, FunctionalState NewState);

// 设置比较值（改变占空比）
void TIM_SetCompare1(TIM_TypeDef* TIMx, uint16_t Compare1);
void TIM_SetCompare2(TIM_TypeDef* TIMx, uint16_t Compare2);
void TIM_SetCompare3(TIM_TypeDef* TIMx, uint16_t Compare3);
void TIM_SetCompare4(TIM_TypeDef* TIMx, uint16_t Compare4);

// 使能定时器
void TIM_Cmd(TIM_TypeDef* TIMx, FunctionalState NewState);

// 使能 PWM 输出（主输出使能，仅高级定时器需要）
void TIM_CtrlPWMOutputs(TIM_TypeDef* TIMx, FunctionalState NewState);
```

### 2.2 输出比较结构体

```c
typedef struct {
    uint16_t TIM_OCMode;        // 输出比较模式
    uint16_t TIM_OutputState;   // 输出使能
    uint16_t TIM_OutputNState;  // 互补输出使能（仅高级定时器）
    uint16_t TIM_Pulse;         // 比较值（CCR）
    uint16_t TIM_OCPolarity;    // 输出极性
    uint16_t TIM_OCNPolarity;   // 互补输出极性
    uint16_t TIM_OCIdleState;   // 空闲状态
    uint16_t TIM_OCNIdleState;  // 互补空闲状态
} TIM_OCInitTypeDef;

// 常用模式
TIM_OCMode_PWM1  // PWM 模式 1：CNT < CCR 时有效
TIM_OCMode_PWM2  // PWM 模式 2：CNT < CCR 时无效

// 输出极性
TIM_OCPolarity_High   // 高电平有效
TIM_OCPolarity_Low    // 低电平有效
```

---

## 三、核心原理

### 3.1 PWM 频率与占空比计算

```
PWM 频率：
    F_pwm = F_clk / ((PSC + 1) × (ARR + 1))

PWM 周期：
    T_pwm = (PSC + 1) × (ARR + 1) / F_clk

占空比：
    Duty = CCR / (ARR + 1) × 100%

示例（1kHz PWM，72MHz 时钟）：
    PSC = 71      → 分频后 1MHz
    ARR = 999     → 计数 1000 次 = 1kHz
    CCR = 500     → 占空比 50%
    CCR = 250     → 占空比 25%
    CCR = 750     → 占空比 75%
```

### 3.2 PWM 模式 1 vs 模式 2

```
PWM 模式 1（向上计数）：
    CNT < CCR  → 输出有效电平
    CNT ≥ CCR  → 输出无效电平

PWM 模式 2（向上计数）：
    CNT < CCR  → 输出无效电平
    CNT ≥ CCR  → 输出有效电平

图示（有效电平为高）：

    PWM1:     ────┐            ┌────
                  │            │
    CNT:    0 ────┴────────────┴────
                  ↑            ↑
                 CCR          ARR
              
    PWM2:     ────────┐    ┌────────
                      │    │
    CNT:    0 ────────┴────┴────────
                      ↑    ↑
                     CCR  ARR

结论：两种模式效果相反，根据需要选择
```

---

## 四、电路连接

### 4.1 硬件清单

| 组件 | 数量 | 说明 |
|------|------|------|
| STM32F103C8T6 最小系统板 | 1 | - |
| LED | 1 | PWM 调光演示 |
| 杜邦线 | 若干 | 连接电路 |

### 4.2 推荐接线方案

| STM32 引脚 | 连接 | 说明 |
|------------|------|------|
| PA0 | LED → GND | TIM2 CH1 PWM 输出 |
| 或 PC13 | 板载 LED | 需要软件反转（低电平亮） |

### 4.3 电路图

```
        STM32F103C8T6
       ┌─────────────┐
       │             │
       │    PA0 ─────┼────┬──── LED
       │  (TIM2_CH1) │    │
       │             │    │
       │             │    R (限流电阻)
       │             │    │
       │    GND ─────┼────┴──── GND
       │             │
       └─────────────┘

注意：
    PA0 输出 PWM 波形，控制 LED 亮度
    如果使用板载 LED（PC13），需要特殊处理
```

---

## 五、完整代码实现

### 5.1 实验一：固定占空比 PWM 输出

```c
#include "stm32f10x.h"

// ==================== PWM 初始化（TIM2 CH1 - PA0）====================
void PWM_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    TIM_TimeBaseInitTypeDef TIM_TimeBaseStructure;
    TIM_OCInitTypeDef TIM_OCInitStructure;

    // 1. 开启时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);

    // 2. 配置 GPIO 为复用推挽输出
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;  // 复用推挽
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // 3. 定时器时基配置（1kHz PWM）
    TIM_TimeBaseStructure.TIM_Period = 999;           // ARR
    TIM_TimeBaseStructure.TIM_Prescaler = 71;         // PSC（1MHz 计数）
    TIM_TimeBaseStructure.TIM_ClockDivision = 0;
    TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInit(TIM2, &TIM_TimeBaseStructure);

    // 4. PWM 模式配置
    TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;       // PWM 模式 1
    TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;
    TIM_OCInitStructure.TIM_Pulse = 500;                     // CCR（50% 占空比）
    TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_High;
    TIM_OC1Init(TIM2, &TIM_OCInitStructure);

    // 5. 使能预装载
    TIM_OC1PreloadConfig(TIM2, TIM_OCPreload_Enable);
    TIM_ARRPreloadConfig(TIM2, ENABLE);

    // 6. 启动定时器
    TIM_Cmd(TIM2, ENABLE);
}

// ==================== 主函数 ====================
int main(void)
{
    PWM_Init();

    while (1)
    {
        // PWM 持续输出，无需干预
    }
}
```

### 5.2 实验二：可调占空比 PWM

```c
#include "stm32f10x.h"

void PWM_Init(void)
{
    // ... 同上 ...
}

// 设置占空比（0-100%）
void PWM_SetDuty(uint8_t duty)
{
    uint16_t ccr;
    if (duty > 100) duty = 100;
    ccr = duty * 10;  // ARR = 999，所以 duty * 10 = CCR
    TIM_SetCompare1(TIM2, ccr);
}

int main(void)
{
    int8_t duty = 0;
    int8_t step = 1;

    PWM_Init();

    while (1)
    {
        // 占空比从 0% 渐变到 100%，再回到 0%
        duty += step;
        if (duty >= 100 || duty <= 0) step = -step;

        PWM_SetDuty(duty);

        // 简单延时
        for (uint32_t i = 0; i < 100000; i++);
    }
}
```

### 5.3 实验三：按键控制亮度

```c
#include "stm32f10x.h"

uint8_t brightness = 50;  // 初始亮度 50%

void PWM_Init(void)
{
    // ... 同实验一 ...
    TIM_SetCompare1(TIM2, 500);  // 初始 50%
}

void GPIO_Config(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

    // PA1, PA2 作为按键输入
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_1 | GPIO_Pin_2;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
}

uint8_t Key_Scan(void)
{
    static uint8_t key_state = 0;

    if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_1) == 0)
    {
        if (key_state == 0)
        {
            key_state = 1;
            return 1;  // KEY1 按下
        }
    }
    else if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_2) == 0)
    {
        if (key_state == 0)
        {
            key_state = 1;
            return 2;  // KEY2 按下
        }
    }
    else
    {
        key_state = 0;
    }
    return 0;
}

void delay_ms(uint32_t ms)
{
    for (uint32_t i = 0; i < ms; i++)
        for (uint32_t j = 0; j < 8000; j++);
}

int main(void)
{
    PWM_Init();
    GPIO_Config();

    while (1)
    {
        uint8_t key = Key_Scan();

        if (key == 1)  // 增加亮度
        {
            if (brightness < 100) brightness += 10;
        }
        else if (key == 2)  // 降低亮度
        {
            if (brightness > 0) brightness -= 10;
        }

        TIM_SetCompare1(TIM2, brightness * 10);
        delay_ms(50);
    }
}
```

---

## 六、调试技巧

### 6.1 常见问题排查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| 无 PWM 输出 | GPIO 模式错误 | 使用 `GPIO_Mode_AF_PP` |
| 占空比不对 | CCR 值计算错误 | 检查 CCR 与 ARR 的关系 |
| 频率不对 | PSC/ARR 配置错误 | 重新计算分频系数 |
| 高级定时器无输出 | 主输出未使能 | 调用 `TIM_CtrlPWMOutputs` |

### 6.2 使用逻辑分析仪测量

```
测量项目：
1. PWM 频率
2. 占空比
3. 高/低电平电压

验证计算是否正确
```

---

## 七、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【PWM 原理】计数器与比较寄存器比较输出             │
│  【占空比】Duty = CCR / (ARR + 1) × 100%           │
│  【频率】F_pwm = F_clk / ((PSC+1) × (ARR+1))       │
│  【GPIO 模式】复用推挽输出 GPIO_Mode_AF_PP          │
│  【调整占空比】TIM_SetCompareX() 动态修改 CCR       │
└─────────────────────────────────────────────────────┘
```

### 初始化流程图

```
    开始
      │
      ↓
┌─────────────────────┐
│ 开启 TIM 和 GPIO    │
│ 时钟                │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ GPIO 配置为         │
│ 复用推挽输出        │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ TIM_TimeBaseInit    │  ← 配置 PSC、ARR
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ TIM_OCInit          │  ← 配置 PWM 模式、CCR
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ 使能预装载          │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ TIM_Cmd 启动定时器  │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ PWM 持续输出        │
└─────────────────────┘
```

### 记忆口诀

```
PWM 本质是比较，计数器比 CCR。
CNT 小了输出高，大了输出低。
占空比是 CCR 除 ARR，频率分频算仔细。
```

---

## 八、拓展练习

1. **呼吸灯预备**：实现占空比自动渐变（为 Day 09 做准备）
2. **多路 PWM**：同时输出 4 路 PWM，每路占空比不同
3. **频率可调**：通过按键调节 PWM 频率
4. **PWM 测量**：使用逻辑分析仪验证频率和占空比

---

*上一篇：Day 07 - 定时器中断*  
*下一篇：Day 09 - PWM 呼吸灯*