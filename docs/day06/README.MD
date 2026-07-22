# Day 06 - 外部中断学习笔记

> 学习日期：2026-07-22

---

## 一、核心知识点

### 1.1 什么是中断？

#### 类比理解

想象你在看书（执行主程序），这时候：

```
【无中断模式】
你：一直看书，哪怕有人敲门你也听不见
结果：门外的快递员等了很久，最后走了

【中断模式】
你：正在看书
门铃响了（中断请求）
你：放下书，去开门（响应中断）
处理完快递（执行中断服务函数）
你：回来继续看书（返回主程序）
```

#### 专业定义

| 概念 | 说明 |
|------|------|
| **中断** | CPU 暂停当前程序，转去处理紧急事件，处理完后返回继续执行 |
| **中断源** | 触发中断的事件来源（如按键、定时器、串口） |
| **中断向量** | 中断服务函数的入口地址 |
| **中断优先级** | 多个中断同时发生时，决定谁先执行 |
| **中断嵌套** | 高优先级中断可以打断低优先级中断 |

### 1.2 STM32 的中断系统

#### NVIC（嵌套向量中断控制器）

```
        外部中断源                    NVIC                     CPU
    ┌─────────────┐            ┌─────────────┐         ┌─────────┐
    │ 按键（EXTI） │───────────▶│             │         │         │
    ├─────────────┤            │  优先级判断  │────────▶│  ARM    │
    │ 定时器（TIM）│───────────▶│  中断屏蔽   │         │ Cortex- │
    ├─────────────┤            │  嵌套管理   │         │  M3     │
    │ 串口（USART）│───────────▶│             │         │         │
    └─────────────┘            └─────────────┘         └─────────┘

NVIC 的作用：
1. 决定哪个中断能被响应（屏蔽）
2. 决定响应顺序（优先级）
3. 支持中断嵌套（高优先级打断低优先级）
```

#### STM32F103 中断资源

| 类型 | 数量 | 说明 |
|------|------|------|
| 内核异常 | 15 | Reset、NMI、HardFault 等 |
| 外部中断 | 60 | EXTI、定时器、串口、ADC 等 |
| 优先级级别 | 16 | 0-15，数值越小优先级越高 |

### 1.3 EXTI（外部中断/事件控制器）

#### EXTI 与 GPIO 的关系

```
                    GPIO 引脚映射到 EXTI 线

     PA0 ──┐
     PB0 ──┼──────────▶ EXTI_Line0（外部中断线 0）
     PC0 ──┘           同一时刻只能选择一个端口

     PA1 ──┐
     PB1 ──┼──────────▶ EXTI_Line1
     PC1 ──┘

     ...（共 16 条 EXTI 线，对应 Pin 0~15）

     ┌──────────────────────────────────────────────┐
     │  重要规则：                                   │
     │  - PA0、PB0、PC0 不能同时使用中断            │
     │  - 因为它们共享 EXTI_Line0                   │
     │  - 只能选择其中一个作为中断源                │
     └──────────────────────────────────────────────┘
```

#### EXTI 触发方式

| 触发方式 | 宏定义 | 说明 | 适用场景 |
|----------|--------|------|----------|
| 上升沿触发 | `EXTI_Trigger_Rising` | 低→高瞬间触发 | 按键释放（上拉） |
| 下降沿触发 | `EXTI_Trigger_Falling` | 高→低瞬间触发 | 按键按下（上拉） |
| 双边沿触发 | `EXTI_Trigger_Rising_Falling` | 两种都触发 | 特殊需求 |

#### 为什么按键用下降沿？

```
上拉输入模式下的按键：

默认状态：高电平（内部上拉）
          ┌───────────────── VCC（内部）
          │
      PA0 ├─────────────── 读到 1

按下瞬间：高→低（下降沿）
          ┌───────────────── VCC
          │
      PA0 ├──────┬──────── 读到 0
                 │
               ┌─┴─┐
               │按键│
               └─┬─┘
                 │
                GND

结论：上拉模式下，按下瞬间是下降沿，释放瞬间是上升沿
```

### 1.4 中断优先级

#### 抢占优先级与响应优先级

```
STM32 使用 4 位（二进制位）来表示优先级：

┌────────────────────────────────────────────────────┐
│  优先级分组（NVIC_PriorityGroup）                  │
│                                                    │
│  Group  │ 抢占位数 │ 响应位数 │ 说明               │
│  ───────┼──────────┼──────────┼───────────────────│
│  0      │    0     │    4     │ 无抢占，16级响应   │
│  1      │    1     │    3     │ 2级抢占，8级响应   │
│  2      │    2     │    2     │ 4级抢占，4级响应   │ ← 最常用
│  3      │    3     │    1     │ 8级抢占，2级响应   │
│  4      │    4     │    0     │ 16级抢占，无响应   │
└────────────────────────────────────────────────────┘

抢占优先级（Preemption Priority）：
    - 数值越小，优先级越高
    - 高抢占优先级可以打断低抢占优先级（嵌套）

响应优先级（Sub Priority）：
    - 抢占优先级相同时，响应优先级高的先执行
    - 但不能嵌套，只能排队

示例（Group 2）：
    ┌─────────────────────────────────────────────────┐
    │  中断A：抢占=0，响应=0  ← 最高优先级            │
    │  中断B：抢占=0，响应=1                          │
    │  中断C：抢占=1，响应=0                          │
    │  中断D：抢占=2，响应=0  ← 最低优先级            │
    │                                                │
    │  执行顺序（同时发生）：A → B → C → D           │
    │  嵌套规则：A 可以打断 B、C、D                  │
    │           C 可以打断 D                         │
    │           B 和 C 不能互相打断                  │
    └─────────────────────────────────────────────────┘
```

### 1.5 中断服务函数命名规则

```c
// STM32 标准库预定义的中断函数名（不能修改）

// EXTI 中断（外部中断线 0-4）
void EXTI0_IRQHandler(void);   // EXTI Line 0（PA0/PB0/PC0）
void EXTI1_IRQHandler(void);   // EXTI Line 1（PA1/PB1/PC1）
void EXTI2_IRQHandler(void);   // EXTI Line 2
void EXTI3_IRQHandler(void);   // EXTI Line 3
void EXTI4_IRQHandler(void);   // EXTI Line 4

// EXTI 中断（外部中断线 5-9 共用一个）
void EXTI9_5_IRQHandler(void);  // EXTI Line 5~9

// EXTI 中断（外部中断线 10-15 共用一个）
void EXTI15_10_IRQHandler(void); // EXTI Line 10~15

// 其他常用中断
void TIM2_IRQHandler(void);     // 定时器 2
void USART1_IRQHandler(void);   // 串口 1
```

---

## 二、电路连接

### 2.1 硬件清单

| 组件 | 数量 | 说明 |
|------|------|------|
| STM32F103C8T6 最小系统板 | 1 | 已板载 LED（PC13） |
| 按键开关 | 1-2 | 外部中断触发源 |
| 杜邦线 | 若干 | 连接电路 |

### 2.2 推荐接线方案

| STM32 引脚 | 连接 | 说明 |
|------------|------|------|
| PA0 | 按键 → GND | 外部中断 0（EXTI0） |
| PC13 | 板载 LED | 低电平点亮 |

### 2.3 电路图

```
        STM32F103C8T6
       ┌─────────────┐
       │             │
       │    PA0 ─────┼────┐
       │    (EXTI0)  │    │
       │             │  ┌─┴─┐
       │             │  │KEY│  ← 按键触发中断
       │             │  └─┬─┘
       │             │    │
       │    GND ─────┼────┘
       │             │
       │    PC13 ────┼────●─── 板载 LED
       │             │
       └─────────────┘

工作原理：
1. PA0 配置为上拉输入 + 外部中断
2. 按键按下 → PA0 变低 → 触发下降沿中断
3. 中断服务函数中翻转 LED
```

---

## 三、完整代码实现

### 3.1 实验一：外部中断控制 LED

**功能：** 按键触发外部中断，在中断中翻转 LED。

```c
#include "stm32f10x.h"

// ==================== GPIO 初始化 ====================
void GPIO_Config(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    // 开启时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC | RCC_APB2Periph_GPIOA |
                           RCC_APB2Periph_AFIO, ENABLE);  // ⚠️ 别忘了 AFIO 时钟！

    // 配置 PC13 为推挽输出（LED）
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOC, &GPIO_InitStructure);

    // 配置 PA0 为上拉输入（按键）
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
}

// ==================== EXTI 初始化 ====================
void EXTI_Config(void)
{
    EXTI_InitTypeDef EXTI_InitStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

    // 1. 连接 GPIO 到 EXTI 线
    GPIO_EXTILineConfig(GPIO_PortSourceGPIOA, GPIO_PinSource0);
    //                     ↑ 端口源           ↑ 引脚源
    // 将 PA0 连接到 EXTI_Line0

    // 2. 配置 EXTI 线
    EXTI_InitStructure.EXTI_Line = EXTI_Line0;              // 中断线 0
    EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;     // 中断模式
    EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Falling; // 下降沿触发
    EXTI_InitStructure.EXTI_LineCmd = ENABLE;               // 使能
    EXTI_Init(&EXTI_InitStructure);

    // 3. 配置 NVIC（中断优先级）
    NVIC_InitStructure.NVIC_IRQChannel = EXTI0_IRQn;        // 中断通道
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1; // 抢占优先级
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 1;      // 响应优先级
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;         // 使能
    NVIC_Init(&NVIC_InitStructure);
}

// ==================== 主函数 ====================
int main(void)
{
    // 1. 设置中断优先级分组（整个程序只需设置一次）
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);

    // 2. 初始化 GPIO
    GPIO_Config();

    // 3. 初始化 EXTI
    EXTI_Config();

    // 4. 默认熄灭 LED
    GPIO_SetBits(GPIOC, GPIO_Pin_13);

    // 5. 主循环（空转，等待中断）
    while (1)
    {
        // 这里什么都不做
        // LED 的翻转完全由中断处理
    }
}

// ==================== 中断服务函数 ====================
/**
  * @brief  EXTI Line 0 中断服务函数
  * @note   函数名必须是 EXTI0_IRQHandler（由启动文件定义）
  */
void EXTI0_IRQHandler(void)
{
    // 1. 判断是否是 EXTI Line 0 触发的中断
    if (EXTI_GetITStatus(EXTI_Line0) != RESET)
    {
        // 2. 翻转 LED
        GPIO_WriteBit(GPIOC, GPIO_Pin_13,
            (BitAction)(1 - GPIO_ReadOutputDataBit(GPIOC, GPIO_Pin_13)));

        // 3. 清除中断标志位（必须！否则会无限进入中断）
        EXTI_ClearITPendingBit(EXTI_Line0);
    }
}
```

### 3.2 实验二：多外部中断

**功能：** 两个按键分别触发不同中断，控制 LED 不同行为。

```c
#include "stm32f10x.h"

void GPIO_Config(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC | RCC_APB2Periph_GPIOA |
                           RCC_APB2Periph_GPIOB | RCC_APB2Periph_AFIO, ENABLE);

    // PC13（LED）
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOC, &GPIO_InitStructure);

    // PA0（按键1 → EXTI0）
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // PB1（按键2 → EXTI1）
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_1;
    GPIO_Init(GPIOB, &GPIO_InitStructure);
}

void EXTI_Config(void)
{
    EXTI_InitTypeDef EXTI_InitStructure;
    NVIC_InitTypeDef NVIC_InitStructure;

    // ===== EXTI0（PA0）=====
    GPIO_EXTILineConfig(GPIO_PortSourceGPIOA, GPIO_PinSource0);

    EXTI_InitStructure.EXTI_Line = EXTI_Line0;
    EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;
    EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Falling;
    EXTI_InitStructure.EXTI_LineCmd = ENABLE;
    EXTI_Init(&EXTI_InitStructure);

    NVIC_InitStructure.NVIC_IRQChannel = EXTI0_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0; // 高优先级
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    // ===== EXTI1（PB1）=====
    GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource1);

    EXTI_InitStructure.EXTI_Line = EXTI_Line1;
    EXTI_Init(&EXTI_InitStructure);

    NVIC_InitStructure.NVIC_IRQChannel = EXTI1_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1; // 低优先级
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
    NVIC_Init(&NVIC_InitStructure);
}

int main(void)
{
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);

    GPIO_Config();
    EXTI_Config();

    GPIO_SetBits(GPIOC, GPIO_Pin_13);

    while (1);
}

// EXTI0 中断：翻转 LED
void EXTI0_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line0) != RESET)
    {
        GPIO_WriteBit(GPIOC, GPIO_Pin_13,
            (BitAction)(1 - GPIO_ReadOutputDataBit(GPIOC, GPIO_Pin_13)));
        EXTI_ClearITPendingBit(EXTI_Line0);
    }
}

// EXTI1 中断：点亮 LED（演示优先级）
void EXTI1_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line1) != RESET)
    {
        GPIO_ResetBits(GPIOC, GPIO_Pin_13);  // 点亮
        EXTI_ClearITPendingBit(EXTI_Line1);
    }
}
```

### 3.3 实验三：中断中的软件消抖

**功能：** 在中断服务函数中实现消抖，防止抖动触发多次中断。

```c
#include "stm32f10x.h"

volatile uint8_t key_pressed = 0;  // 中断标志

void delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 8000; j++);
}

void EXTI0_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line0) != RESET)
    {
        // 方法一：延时消抖（不推荐，中断中不应长时间阻塞）
        // delay_ms(20);
        // if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0) == 0)
        // {
        //     key_pressed = 1;
        // }

        // 方法二：设置标志，在主循环中处理（推荐）
        key_pressed = 1;

        EXTI_ClearITPendingBit(EXTI_Line0);
    }
}

int main(void)
{
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);

    // ... 初始化代码 ...

    while (1)
    {
        if (key_pressed)
        {
            key_pressed = 0;
            delay_ms(20);  // 在主循环中消抖
            if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0) == 0)
            {
                GPIO_WriteBit(GPIOC, GPIO_Pin_13,
                    (BitAction)(1 - GPIO_ReadOutputDataBit(GPIOC, GPIO_Pin_13)));
            }
        }
    }
}
```

---

## 四、进阶：中断与轮询的对比

### 4.1 对比表

| 特性 | 轮询方式（Day 05） | 中断方式（Day 06） |
|------|-------------------|-------------------|
| CPU 占用 | 一直轮询，占用高 | 事件触发，占用低 |
| 响应速度 | 取决于轮询周期 | 即时响应（微秒级） |
| 编程复杂度 | 简单 | 较复杂 |
| 适用场景 | 低速事件、简单系统 | 高速事件、实时系统 |

### 4.2 流程对比图

```
【轮询方式】
┌─────────────────────────────────────────────────────┐
│  while(1)                                          │
│  {                                                 │
│      if (Key_Scan())  // ← 每次循环都要检查        │
│      {                                             │
│          处理按键...                               │
│      }                                             │
│      // 其他任务                                   │
│  }                                                 │
│                                                    │
│  缺点：如果按键在"其他任务"期间按下，会错过检测    │
└─────────────────────────────────────────────────────┘

【中断方式】
┌─────────────────────────────────────────────────────┐
│  while(1)                                          │
│  {                                                 │
│      // 主程序专心做自己的事                        │
│      // 按键按下时，硬件自动通知 CPU                │
│  }                                                 │
│                                                    │
│  void EXTI0_IRQHandler(void)  // ← 硬件自动调用    │
│  {                                                 │
│      处理按键...                                   │
│  }                                                 │
│                                                    │
│  优点：永远不会错过事件，CPU 利用率高              │
└─────────────────────────────────────────────────────┘
```

---

## 五、调试技巧

### 5.1 常见问题排查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| 中断不触发 | AFIO 时钟未开启 | 添加 `RCC_APB2Periph_AFIO` |
| 中断只触发一次 | 未清除中断标志 | 调用 `EXTI_ClearITPendingBit` |
| 中断函数名错误 | 函数名不匹配 | 使用标准名称 `EXTI0_IRQHandler` |
| GPIO 与 EXTI 不对应 | 映射错误 | 检查 `GPIO_EXTILineConfig` |
| 优先级冲突 | 分组设置错误 | 确认 `NVIC_PriorityGroupConfig` |

### 5.2 调试方法

```c
// 在中断函数中加入计数器
volatile uint32_t interrupt_count = 0;

void EXTI0_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line0) != RESET)
    {
        interrupt_count++;  // 在调试器中观察此变量
        EXTI_ClearITPendingBit(EXTI_Line0);
    }
}
```

---

## 六、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【中断本质】事件驱动的程序跳转                      │
│  【EXTI 映射】PA0/PB0/PC0 共享 EXTI_Line0           │
│  【触发方式】上拉按键用下降沿触发                    │
│  【优先级规则】数值越小，优先级越高                  │
│  【清除标志】中断服务函数必须清除中断标志位          │
└─────────────────────────────────────────────────────┘
```

### 初始化流程图

```
    开始
      │
      ↓
┌─────────────────────┐
│ NVIC_PriorityGroup  │  ← 设置优先级分组
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ 开启 GPIO + AFIO    │  ← AFIO 时钟容易忘！
│ 时钟                │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ 配置 GPIO 为输入    │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ GPIO_EXTILineConfig │  ← GPIO → EXTI 映射
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ EXTI_Init           │  ← 配置中断线、触发方式
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ NVIC_Init           │  ← 配置中断通道、优先级
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ 主循环等待中断      │
└─────────────────────┘
```

### 记忆口诀

```
外部中断用 EXTI，GPIO 映射别忘记。
下降沿触发按键按，清除标志在结尾。
```

---

## 七、拓展练习

1. **中断计数器**：用中断实现按键按下计数，通过 LED 闪烁显示计数值
2. **双边沿触发**：配置为双边沿触发，分别处理按下和释放事件
3. **中断嵌套实验**：设置两个不同优先级的中断，观察嵌套行为
4. **长按检测**：在中断中记录时间，区分长按和短按

---

*下一篇：Day 07 - 定时器中断*