# Day 06 - 外部中断 EXTI 学习笔记

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
```

---

## 二、常用库函数

### 2.1 NVIC 相关函数

```c
// 设置中断优先级分组（整个程序只需调用一次）
void NVIC_PriorityGroupConfig(uint32_t NVIC_PriorityGroup);
// 参数：NVIC_PriorityGroup_0 ~ NVIC_PriorityGroup_4

// 初始化中断通道
void NVIC_Init(NVIC_InitTypeDef* NVIC_InitStruct);

// 使能/禁用中断通道
void NVIC_EnableIRQ(IRQn_Type IRQn);
void NVIC_DisableIRQ(IRQn_Type IRQn);

// 设置/清除中断挂起位
void NVIC_SetPendingIRQ(IRQn_Type IRQn);
void NVIC_ClearPendingIRQ(IRQn_Type IRQn);
```

### 2.2 EXTI 相关函数

```c
// 将 GPIO 引脚映射到 EXTI 线
void GPIO_EXTILineConfig(uint8_t GPIO_PortSource, uint8_t GPIO_PinSource);
// 示例：GPIO_EXTILineConfig(GPIO_PortSourceGPIOA, GPIO_PinSource0);

// 初始化 EXTI 线
void EXTI_Init(EXTI_InitTypeDef* EXTI_InitStruct);

// 获取中断标志状态
ITStatus EXTI_GetITStatus(uint32_t EXTI_Line);
// 返回：SET 或 RESET

// 清除中断挂起标志
void EXTI_ClearITPendingBit(uint32_t EXTI_Line);

// 生成软件中断
void EXTI_GenerateSWInterrupt(uint32_t EXTI_Line);
```

### 2.3 结构体定义

```c
// NVIC 初始化结构体
typedef struct {
    uint8_t NVIC_IRQChannel;                    // 中断通道（如 EXTI0_IRQn）
    uint8_t NVIC_IRQChannelPreemptionPriority;  // 抢占优先级（0-15）
    uint8_t NVIC_IRQChannelSubPriority;         // 响应优先级（0-15）
    FunctionalState NVIC_IRQChannelCmd;         // 使能/禁用
} NVIC_InitTypeDef;

// EXTI 初始化结构体
typedef struct {
    uint32_t EXTI_Line;               // 中断线（EXTI_Line0 ~ EXTI_Line15）
    EXTIMode_TypeDef EXTI_Mode;       // 模式：Interrupt 或 Event
    EXTITrigger_TypeDef EXTI_Trigger; // 触发方式：Rising/Falling/Both
    FunctionalState EXTI_LineCmd;     // 使能/禁用
} EXTI_InitTypeDef;
```

---

## 三、核心原理

### 3.1 中断响应流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      中断响应完整流程                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 外部事件发生（如按键按下）                                  │
│         ↓                                                       │
│  2. 硬件检测到触发条件（如下降沿）                              │
│         ↓                                                       │
│  3. EXTI 控制器产生中断请求                                     │
│         ↓                                                       │
│  4. NVIC 判断优先级，决定是否响应                               │
│         ↓                                                       │
│  5. CPU 保存当前上下文（PC、寄存器等）                          │
│         ↓                                                       │
│  6. CPU 跳转到中断向量地址                                      │
│         ↓                                                       │
│  7. 执行中断服务函数（ISR）                                     │
│         ↓                                                       │
│  8. 清除中断标志位                                              │
│         ↓                                                       │
│  9. CPU 恢复上下文，返回主程序                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 为什么必须清除中断标志？

```
问题：不清除中断标志会发生什么？

答：中断标志位是"记忆"，它告诉 CPU"刚才有事情发生了"。
    如果你处理完了却不擦掉这个记忆，CPU 会认为"事情还没处理完"，
    然后反复进入中断函数，导致程序卡死。

比喻：
    中断标志 = 未读消息提示
    中断服务函数 = 阅读消息
    
    阅读完消息后，必须把"未读"标记改成"已读"。
    否则系统会一直提示你有新消息。
```

---

## 四、电路连接

### 4.1 硬件清单

| 组件 | 数量 | 说明 |
|------|------|------|
| STM32F103C8T6 最小系统板 | 1 | 已板载 LED（PC13） |
| 按键开关 | 1-2 | 外部中断触发源 |
| 杜邦线 | 若干 | 连接电路 |

### 4.2 推荐接线方案

| STM32 引脚 | 连接 | 说明 |
|------------|------|------|
| PA0 | 按键 → GND | 外部中断 0（EXTI0） |
| PC13 | 板载 LED | 低电平点亮 |

### 4.3 电路图

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

## 五、完整代码实现

### 5.1 实验一：外部中断控制 LED

**功能：** 按键触发外部中断，在中断中翻转 LED。

```c
#include "stm32f10x.h"

// ==================== GPIO 初始化 ====================
void GPIO_Config(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    // 开启时钟（必须包含 AFIO 时钟！）
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC | RCC_APB2Periph_GPIOA |
                           RCC_APB2Periph_AFIO, ENABLE);

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

    // 2. 配置 EXTI 线
    EXTI_InitStructure.EXTI_Line = EXTI_Line0;
    EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;
    EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Falling;
    EXTI_InitStructure.EXTI_LineCmd = ENABLE;
    EXTI_Init(&EXTI_InitStructure);

    // 3. 配置 NVIC
    NVIC_InitStructure.NVIC_IRQChannel = EXTI0_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 1;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);
}

// ==================== 主函数 ====================
int main(void)
{
    // 1. 设置中断优先级分组
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);

    // 2. 初始化
    GPIO_Config();
    EXTI_Config();

    // 3. 默认熄灭 LED
    GPIO_SetBits(GPIOC, GPIO_Pin_13);

    // 4. 主循环等待中断
    while (1)
    {
        // CPU 空闲，可以做其他事
    }
}

// ==================== 中断服务函数 ====================
void EXTI0_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line0) != RESET)
    {
        // 翻转 LED
        GPIO_WriteBit(GPIOC, GPIO_Pin_13,
            (BitAction)(1 - GPIO_ReadOutputDataBit(GPIOC, GPIO_Pin_13)));

        // 清除中断标志（必须！）
        EXTI_ClearITPendingBit(EXTI_Line0);
    }
}
```

### 5.2 实验二：多外部中断

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

    // EXTI0（PA0）- 高优先级
    GPIO_EXTILineConfig(GPIO_PortSourceGPIOA, GPIO_PinSource0);
    EXTI_InitStructure.EXTI_Line = EXTI_Line0;
    EXTI_InitStructure.EXTI_Mode = EXTI_Mode_Interrupt;
    EXTI_InitStructure.EXTI_Trigger = EXTI_Trigger_Falling;
    EXTI_InitStructure.EXTI_LineCmd = ENABLE;
    EXTI_Init(&EXTI_InitStructure);

    NVIC_InitStructure.NVIC_IRQChannel = EXTI0_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 0;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);

    // EXTI1（PB1）- 低优先级
    GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource1);
    EXTI_InitStructure.EXTI_Line = EXTI_Line1;
    EXTI_Init(&EXTI_InitStructure);

    NVIC_InitStructure.NVIC_IRQChannel = EXTI1_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;
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

// EXTI1 中断：点亮 LED
void EXTI1_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line1) != RESET)
    {
        GPIO_ResetBits(GPIOC, GPIO_Pin_13);
        EXTI_ClearITPendingBit(EXTI_Line1);
    }
}
```

---

## 六、调试技巧

### 6.1 常见问题排查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| 中断不触发 | AFIO 时钟未开启 | 添加 `RCC_APB2Periph_AFIO` |
| 中断只触发一次 | 未清除中断标志 | 调用 `EXTI_ClearITPendingBit` |
| 中断函数名错误 | 函数名不匹配 | 使用标准名称 `EXTI0_IRQHandler` |
| GPIO 与 EXTI 不对应 | 映射错误 | 检查 `GPIO_EXTILineConfig` |

### 6.2 调试方法

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

## 七、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【中断本质】事件驱动的程序跳转                      │
│  【EXTI 映射】PA0/PB0/PC0 共享 EXTI_Line0           │
│  【触发方式】上拉按键用下降沿触发                    │
│  【优先级规则】数值越小，优先级越高                  │
│  【清除标志】中断服务函数必须清除中断标志位          │
│  【AFIO 时钟】使用 EXTI 必须开启 AFIO 时钟           │
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
AFIO 时钟必须开，下降沿触按键按。
中断函数要清标，不然程序跑不了。
```

---

## 八、拓展练习

1. **中断计数器**：用中断实现按键按下计数，通过 LED 闪烁显示计数值
2. **双边沿触发**：配置为双边沿触发，分别处理按下和释放事件
3. **中断嵌套实验**：设置两个不同优先级的中断，观察嵌套行为
4. **软件中断**：使用 `EXTI_GenerateSWInterrupt` 触发软件中断

---

*下一篇：Day 07 - 定时器中断*