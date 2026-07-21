# Day 05 - 按键控制 LED 学习笔记

> 学习日期：2026-07-21

---

## 一、核心知识点

### 1.1 GPIO 输入模式

STM32 的 GPIO 有四种输入模式：

| 模式 | 宏定义 | 说明 | 典型用途 |
|------|--------|------|----------|
| 浮空输入 | `GPIO_Mode_IN_FLOATING` | 无上下拉，电平不确定 | 外部已有上下拉电路 |
| 上拉输入 | `GPIO_Mode_IPU` | 内部接 VCC，默认高电平 | 按键接 GND |
| 下拉输入 | `GPIO_Mode_IPD` | 内部接 GND，默认低电平 | 按键接 VCC |
| 模拟输入 | `GPIO_Mode_AIN` | 连接到 ADC | 读取模拟电压 |

**本实验使用上拉输入（IPU）：**
```
按键接 GND → 默认高电平 → 按下变低电平

        VCC (内部)
          │
      ┌───┴───┐
      │ 内部  │
      │ 上拉  │  ← STM32 内部集成
      │ 电阻  │
      └───┬───┘
          │
    PA0 ──┼────→ 读到高电平（默认）
          │
       ┌──┴──┐
       │按键 │
       └──┬──┘
          │
         GND    ← 按下时接地
```

### 1.2 按键消抖原理

#### 为什么需要消抖？

按键内部的金属触点在接触或断开瞬间会产生**机械抖动**：

```
按键信号真实波形：

      ┌──┐ ┌┐ ┌──┐                ┌─┐ ┌──┐
      │  │ ││ │  │                │ │ │  │
──────┘  └─┘└─┘  └────────────────┘ └─┘  └──────
      ↑        ↑                  ↑
    前沿抖动  稳定期            后沿抖动
    (5-10ms)                    (5-10ms)

不消抖的后果：
- 一次按键可能被识别为多次
- LED 状态随机翻转
- 计数器不准确
```

#### 软件消抖方法

| 方法 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **延时法** | 检测到变化后延时 20ms 再确认 | 简单易实现 | 阻塞 CPU |
| **计数法** | 定时器每 2ms 采样，连续 10 次一致才确认 | 非阻塞，精确 | 需要定时器 |
| **状态机** | 用状态机管理按键生命周期 | 结构清晰，可扩展 | 代码量大 |

#### 状态机消抖原理

**核心思想：** 把按键看作一个有"状态"的对象，而不是简单的电平检测。

```
状态转换图：

         检测到低电平
         连续 10 次
    ┌─────────────────────┐
    │                     │
    │                     ↓
┌───┴───┐           ┌─────┴─────┐
│ 释放  │           │   按下    │
│ 状态  │           │   状态    │
└───┬───┘           └─────┬─────┘
    ↑                     │
    │    检测到高电平     │
    │    连续 10 次       │
    └─────────────────────┘

状态机 vs 延时法的区别：

┌────────────────────────────────────────────────────────────┐
│  延时法（阻塞式）                                          │
│  ──────────────                                            │
│  CPU: ──检测──等待20ms──再检测──等待释放──等待20ms──执行    │
│                    ↑                                       │
│            这段时间 CPU 什么都做不了                        │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│  状态机（非阻塞式）                                        │
│  ────────────────                                          │
│  CPU: ──检测(2ms)──做其他事──检测(2ms)──做其他事──...       │
│        ↑         ↑         ↑                               │
│     定时器中断  主循环    定时器中断                         │
│                                                            │
│  每 2ms "看一眼" 按键状态，连续 10 次一致才确认              │
│  期间 CPU 可以做其他任务                                    │
└────────────────────────────────────────────────────────────┘
```

**为什么定时器采样间隔是 2ms？**

```
采样间隔 × 连续次数 = 消抖时间

2ms × 10 次 = 20ms（刚好覆盖抖动期 5-10ms）

采样太慢（如 20ms × 1 次）→ 可能错过短按
采样太快（如 1ms × 20 次）→ 浪费 CPU 资源
```

**状态机消抖伪代码：**

```c
// 每 2ms 调用一次（在定时器中断中）
void Key_FSM(void)
{
    static uint8_t count = 0;
    static uint8_t state = RELEASE;

    switch (state)
    {
        case RELEASE:  // 当前是释放状态
            if (按键 == 低电平)
            {
                count++;
                if (count >= 10)  // 连续 10 次低电平
                {
                    state = PRESS;
                    pressed = 1;   // 通知主程序
                }
            }
            else
            {
                count = 0;  // 不一致，重置计数
            }
            break;

        case PRESS:  // 当前是按下状态
            if (按键 == 高电平)
            {
                count++;
                if (count >= 10)  // 连续 10 次高电平
                {
                    state = RELEASE;
                }
            }
            else
            {
                count = 0;
            }
            break;
    }
}
```

### 1.3 常用库函数速查

#### GPIO 输入相关

```c
// 读取单个引脚电平（返回 0 或 1）
uint8_t GPIO_ReadInputDataBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);

// 读取整个端口（返回 16 位值）
uint16_t GPIO_ReadInputData(GPIO_TypeDef* GPIOx);

// 读取输出数据寄存器（用于读取当前输出状态）
uint8_t GPIO_ReadOutputDataBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);
```

#### GPIO 输出相关

```c
// 置位（输出高电平）
void GPIO_SetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);

// 复位（输出低电平）
void GPIO_ResetBits(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin);

// 写入指定值
void GPIO_WriteBit(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin, BitAction BitVal);

// 写入整个端口
void GPIO_Write(GPIO_TypeDef* GPIOx, uint16_t PortVal);
```

#### GPIO 初始化

```c
// 开启时钟
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOx, ENABLE);

// 配置 GPIO
GPIO_InitTypeDef GPIO_InitStructure;
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_x;           // 引脚
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;       // 模式
GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;   // 速度（输出模式有效）
GPIO_Init(GPIOx, &GPIO_InitStructure);
```

### 1.4 按键扫描函数模板

```c
/**
  * @brief  按键扫描（带消抖）
  * @param  GPIOx: GPIO 端口（如 GPIOA）
  * @param  GPIO_Pin: 引脚号（如 GPIO_Pin_0）
  * @retval 1 = 按下，0 = 未按下
  * @note   使用内部上拉，按下为低电平
  * 
  * 流程：检测 → 延时消抖 → 再确认 → 等释放 → 消抖
  */
uint8_t Key_Scan(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)
{
    // 第一步：检测是否为低电平
    if (GPIO_ReadInputDataBit(GPIOx, GPIO_Pin) == 0)
    {
        // 第二步：延时 20ms，跳过前沿抖动
        delay_ms(20);

        // 第三步：再次检测，确认是否真的按下
        if (GPIO_ReadInputDataBit(GPIOx, GPIO_Pin) == 0)
        {
            // 第四步：等待按键释放（阻塞）
            while (GPIO_ReadInputDataBit(GPIOx, GPIO_Pin) == 0);
            
            // 第五步：延时 20ms，消除后沿抖动
            delay_ms(20);

            return 1;  // 确认按键按下
        }
    }
    return 0;  // 按键未按下或只是抖动
}
```

**消抖时序图：**

```
检测点1     检测点2              检测点3
    ↓          ↓                    ↓
────┼──────────┼────────────────────┼────
    │← 延时 → │                    │← 延时
     20ms                            20ms

条件：检测点1 和 检测点2 都为低电平 → 确认按下
```

---

## 二、电路连接

### 2.1 硬件清单

| 组件 | 数量 | 说明 |
|------|------|------|
| STM32F103C8T6 最小系统板 | 1 | 已板载 LED（PC13） |
| 按键开关 | 1-4 | 四脚直插按键或两脚轻触开关 |
| 杜邦线 | 若干 | 连接电路 |
| 面包板 | 1 | 搭建电路 |

### 2.2 推荐接线方案（内部上拉）

| STM32 引脚 | 连接 | 说明 |
|------------|------|------|
| PA0 | 按键 → GND | 按键1（内部上拉） |
| PA1 | 按键 → GND | 按键2（内部上拉） |
| PC13 | 板载 LED | 低电平点亮 |

**电路图：**

```
        STM32F103C8T6
       ┌─────────────┐
       │             │
       │    PA0 ─────┼────┐
       │             │    │
       │             │  ┌─┴─┐
       │             │  │KEY│  ← 按键
       │             │  └─┬─┘
       │             │    │
       │    GND ─────┼────┘
       │             │
       │    PC13 ────┼────●─── 板载 LED
       │             │
       └─────────────┘

工作原理：
- GPIO 配置为内部上拉输入（GPIO_Mode_IPU）
- 按键一端接 GPIO，另一端接 GND
- 默认状态：GPIO 读到高电平（内部上拉）
- 按下状态：GPIO 读到低电平（直接接地）
```

**实际接线示意图：**

```
    STM32 开发板                    面包板
   ┌───────────┐                  ┌─────────┐
   │           │                  │         │
   │   PA0  ───┼──────────────────┼──┬──────┤
   │           │                  │  │      │
   │   GND  ───┼──────────────────┼──┴──────┤ ← 按键跨接 PA0 和 GND
   │           │                  │         │
   │   PC13 ───┼── 板载LED ───────│         │
   │           │                  │         │
   └───────────┘                  └─────────┘
```

---

## 三、完整代码实现

### 3.1 实验一：按键控制 LED 开关

**功能：** 按一下亮，再按一下灭。

```c
#include "stm32f10x.h"

// ==================== 延时函数 ====================
void delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 8000; j++);  // 72MHz 下约 1ms
}

// ==================== 按键扫描函数 ====================
uint8_t Key_Scan(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)
{
    if (GPIO_ReadInputDataBit(GPIOx, GPIO_Pin) == 0)
    {
        delay_ms(20);  // 消抖延时
        if (GPIO_ReadInputDataBit(GPIOx, GPIO_Pin) == 0)
        {
            while (GPIO_ReadInputDataBit(GPIOx, GPIO_Pin) == 0);  // 等待释放
            delay_ms(20);  // 释放消抖
            return 1;
        }
    }
    return 0;
}

// ==================== GPIO 初始化 ====================
void GPIO_Config(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    // 开启 GPIOC 和 GPIOA 时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC | RCC_APB2Periph_GPIOA, ENABLE);

    // 配置 PC13 为推挽输出（LED）
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOC, &GPIO_InitStructure);

    // 配置 PA0 为上拉输入（按键）
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;  // Input Pull-Up
    GPIO_Init(GPIOA, &GPIO_InitStructure);
}

// ==================== 主函数 ====================
int main(void)
{
    GPIO_Config();
    GPIO_SetBits(GPIOC, GPIO_Pin_13);  // 默认 LED 熄灭

    while (1)
    {
        if (Key_Scan(GPIOA, GPIO_Pin_0))
        {
            // 翻转 LED 状态
            // (BitAction)(1 - x)：如果是1变成0，如果是0变成1
            GPIO_WriteBit(GPIOC, GPIO_Pin_13,
                (BitAction)(1 - GPIO_ReadOutputDataBit(GPIOC, GPIO_Pin_13)));
        }
    }
}
```

### 3.2 实验二：双按键控制 LED

**功能：** KEY1 点亮 LED，KEY2 熄灭 LED。

```c
#include "stm32f10x.h"

void delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 8000; j++);
}

uint8_t Key_Scan(GPIO_TypeDef* GPIOx, uint16_t GPIO_Pin)
{
    if (GPIO_ReadInputDataBit(GPIOx, GPIO_Pin) == 0)
    {
        delay_ms(20);
        if (GPIO_ReadInputDataBit(GPIOx, GPIO_Pin) == 0)
        {
            while (GPIO_ReadInputDataBit(GPIOx, GPIO_Pin) == 0);
            delay_ms(20);
            return 1;
        }
    }
    return 0;
}

void GPIO_Config(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC | RCC_APB2Periph_GPIOA, ENABLE);

    // PC13（LED）推挽输出
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOC, &GPIO_InitStructure);

    // PA0、PA1（按键）上拉输入
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0 | GPIO_Pin_1;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStructure);
}

int main(void)
{
    GPIO_Config();
    GPIO_SetBits(GPIOC, GPIO_Pin_13);

    while (1)
    {
        if (Key_Scan(GPIOA, GPIO_Pin_0))  // KEY1 → 点亮
        {
            GPIO_ResetBits(GPIOC, GPIO_Pin_13);
        }

        if (Key_Scan(GPIOA, GPIO_Pin_1))  // KEY2 → 熄灭
        {
            GPIO_SetBits(GPIOC, GPIO_Pin_13);
        }
    }
}
```

### 3.3 实验三：长按短按区分

**功能：** 短按翻转 LED，长按（>2秒）快速闪烁 3 次。

```c
#include "stm32f10x.h"

void delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 8000; j++);
}

/**
  * @brief  按键扫描（区分长按短按）
  * @retval 0 = 未按下，1 = 短按，2 = 长按
  */
uint8_t Key_Scan_LongShort(void)
{
    uint16_t press_time = 0;

    if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0) == 0)
    {
        delay_ms(20);
        if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0) == 0)
        {
            // 计时，直到释放或超过 2 秒
            while (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0) == 0 && press_time < 200)
            {
                delay_ms(10);
                press_time++;
            }

            while (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0) == 0);
            delay_ms(20);

            return (press_time >= 200) ? 2 : 1;
        }
    }
    return 0;
}

int main(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    uint8_t key_result, i;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC | RCC_APB2Periph_GPIOA, ENABLE);

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_13;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOC, &GPIO_InitStructure);

    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    GPIO_SetBits(GPIOC, GPIO_Pin_13);

    while (1)
    {
        key_result = Key_Scan_LongShort();

        if (key_result == 1)  // 短按：翻转
        {
            GPIO_WriteBit(GPIOC, GPIO_Pin_13,
                (BitAction)(1 - GPIO_ReadOutputDataBit(GPIOC, GPIO_Pin_13)));
        }
        else if (key_result == 2)  // 长按：闪烁 3 次
        {
            for (i = 0; i < 6; i++)
            {
                GPIO_WriteBit(GPIOC, GPIO_Pin_13,
                    (BitAction)(1 - GPIO_ReadOutputDataBit(GPIOC, GPIO_Pin_13)));
                delay_ms(100);
            }
        }
    }
}
```

---

## 四、进阶：状态机消抖

### 4.1 为什么需要状态机？

阻塞式 `while(...);` 会卡住 CPU，无法同时做其他事情（如读取传感器、刷新显示屏）。

### 4.2 状态机实现

```c
// 按键状态定义
typedef enum {
    KEY_STATE_RELEASE,  // 释放状态
    KEY_STATE_PRESS     // 按下状态
} Key_State;

// 按键结构体
typedef struct {
    GPIO_TypeDef* GPIOx;
    uint16_t GPIO_Pin;
    Key_State state;
    uint8_t count;
    uint8_t pressed;
} Key_TypeDef;

Key_TypeDef key1 = {GPIOA, GPIO_Pin_0, KEY_STATE_RELEASE, 0, 0};

/**
  * @brief  状态机消抖（每 2ms 调用一次）
  */
void Key_FSM(Key_TypeDef* key)
{
    uint8_t level = GPIO_ReadInputDataBit(key->GPIOx, key->GPIO_Pin);

    switch (key->state)
    {
        case KEY_STATE_RELEASE:
            if (level == 0)  // 检测到低电平
            {
                key->count++;
                if (key->count >= 10)  // 连续 10 次 = 20ms
                {
                    key->state = KEY_STATE_PRESS;
                    key->pressed = 1;
                }
            }
            else
            {
                key->count = 0;
            }
            break;

        case KEY_STATE_PRESS:
            if (level == 1)  // 检测到高电平（释放）
            {
                key->count++;
                if (key->count >= 10)
                {
                    key->state = KEY_STATE_RELEASE;
                }
            }
            else
            {
                key->count = 0;
            }
            break;
    }
}

// 在定时器中断中调用（每 2ms）
void TIM2_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM2, TIM_IT_Update) != RESET)
    {
        TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
        Key_FSM(&key1);
    }
}

// 主循环（非阻塞）
int main(void)
{
    // ... 初始化代码 ...

    while (1)
    {
        if (key1.pressed)
        {
            key1.pressed = 0;
            // 执行按键动作
        }

        // 其他任务不会被阻塞
    }
}
```

---

## 五、调试技巧

### 5.1 验证消抖是否成功

**方法一：计数测试**
```c
uint8_t count = 0;
while (1) {
    if (Key_Scan(GPIOA, GPIO_Pin_0)) {
        count++;
        // 显示 count 值
    }
}
```
按一次，计数应该只加 1。

**方法二：串口打印**
```c
printf("Key pressed: %d\n", count++);
```
每次按键只应打印一行。

### 5.2 常见问题排查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| 按键无反应 | 引脚配置错误 | 检查 `GPIO_Mode_IPU` |
| 按键多次触发 | 消抖时间太短 | 增加 `delay_ms` 到 20-50ms |
| 按键灵敏度低 | 消抖时间太长 | 减少 `delay_ms` 到 10-20ms |
| LED 不亮 | 接线错误 | 检查是高电平还是低电平点亮 |

---

## 六、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【输入模式】GPIO_Mode_IPU = 上拉输入              │
│  【消抖时间】20ms 是常用值                          │
│  【扫描流程】检测 → 延时 → 再检测 → 等释放         │
│  【LED 翻转】WriteBit(pin, 1 - ReadOutputDataBit)   │
└─────────────────────────────────────────────────────┘
```

### 流程图

```
    开始
      │
      ↓
┌─────────────┐
│ 初始化 GPIO │
│ LED = 输出  │
│ KEY = IPU   │
└──────┬──────┘
       ↓
┌─────────────┐
│  主循环     │←───────────┐
└──────┬──────┘            │
       ↓                   │
   按键按下？──否──────────┘
       │是
       ↓
┌─────────────┐
│ 翻转 LED    │
└─────────────┘
```

### 记忆口诀

```
上拉输入默认高，按键接地按下低。
消抖延时二十毫，再测确认不多余。
```

---

## 七、拓展练习

1. **双击检测**：检测连续两次快速按下（间隔 < 500ms）
2. **组合键**：同时按下两个按键触发特殊功能
3. **按键密码**：按特定序列（如：短-短-长）触发动作
4. **呼吸灯控制**：按键控制 LED 亮度渐变

---

*下一篇：Day 06 - 外部中断*