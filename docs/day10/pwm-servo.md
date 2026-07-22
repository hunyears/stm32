# Day 10 - PWM 舵机控制学习笔记

> 学习日期：2026-07-26

---

## 一、核心知识点

### 1.1 什么是舵机？

#### 定义

舵机（Servo Motor）是一种位置伺服驱动器，可以精确控制旋转角度。

```
舵机外观：

    ┌────────────────┐
    │    ┌──────┐    │ ← 输出轴
    │    │  ○   │    │
    │    └──────┘    │
    │                │
    │  SG90 舵机     │
    │                │
    └──┬───┬───┬───┘
       │   │   │
       棕 红 橙
       │   │   │
      GND VCC 信号
```

#### 舵机类型

| 类型 | 角度范围 | 扭力 | 价格 | 适用场景 |
|------|----------|------|------|----------|
| SG90 | 0°~180° | 小 | 低 | 学习、小项目 |
| MG996R | 0°~180° | 大 | 中 | 机器人、机械臂 |
| 连续旋转舵机 | 无限旋转 | - | - | 小车驱动 |

### 1.2 舵机的控制原理

#### PWM 控制信号

```
舵机控制 PWM 要求：
    - 频率：50Hz（周期 20ms）
    - 高电平宽度：0.5ms ~ 2.5ms 对应 0° ~ 180°

时序图：

        ┌───┐               ┌───┐
        │   │               │   │
    ────┘   └───────────────┘   └────────
        │←─→│
        高电平宽度

角度对应表：
┌───────────────────────────────────────────────────┐
│  角度   │  高电平时间  │    CCR 值（ARR=20000）   │
├─────────┼──────────────┼─────────────────────────│
│   0°    │    0.5ms     │        500               │
│  45°    │    1.0ms     │       1000               │
│  90°    │    1.5ms     │       1500               │
│  135°   │    2.0ms     │       2000               │
│  180°   │    2.5ms     │       2500               │
└───────────────────────────────────────────────────┘

计算公式：
    CCR = 500 + angle × (2000 / 180)
    CCR = 500 + angle × 11.11
```

#### 舵机内部工作原理

```
┌────────────────────────────────────────────────────────┐
│                    舵机内部控制                         │
│                                                        │
│  PWM输入 ──┬───▶ 比较器 ──▶ 电机驱动 ──▶ 电机         │
│            │                                           │
│            │    ┌─────┐                                │
│            └───▶│电位器│◀─── 反馈                      │
│                 └─────┘                                │
│                                                        │
│  工作流程：                                            │
│  1. PWM 信号指定目标角度                               │
│  2. 电位器检测当前角度                                 │
│  3. 比较目标与当前角度                                 │
│  4. 驱动电机转向目标角度                               │
│  5. 到达目标后停止                                     │
└────────────────────────────────────────────────────────┘
```

### 1.3 舵机与普通电机的区别

| 特性 | 舵机 | 普通电机 |
|------|------|----------|
| 控制 | 精确角度 | 转速/方向 |
| 反馈 | 有（内部电位器） | 无 |
| 位置保持 | 能 | 不能 |
| 运动范围 | 有限（通常 180°） | 无限旋转 |
| 应用 | 机械臂、云台 | 风扇、车轮 |

---

## 二、常用库函数

### 2.1 PWM 相关函数（复习）

```c
// 定时器时基初始化
void TIM_TimeBaseInit(TIM_TypeDef* TIMx, TIM_TimeBaseInitTypeDef* TIM_TimeBaseInitStruct);

// 输出比较初始化
void TIM_OCInit(TIM_TypeDef* TIMx, TIM_OCInitTypeDef* TIM_OCInitStruct);

// 设置比较值
void TIM_SetCompare1(TIM_TypeDef* TIMx, uint16_t Compare1);

// 使能定时器
void TIM_Cmd(TIM_TypeDef* TIMx, FunctionalState NewState);
```

### 2.2 舵机控制函数

```c
// 设置舵机角度（0-180°）
void Servo_SetAngle(uint8_t angle)
{
    uint16_t ccr;
    if (angle > 180) angle = 180;
    
    // CCR = 500 + angle * (2000 / 180)
    ccr = 500 + (uint32_t)angle * 2000 / 180;
    TIM_SetCompare1(TIM2, ccr);
}
```

---

## 三、核心原理

### 3.1 PWM 频率配置

```
舵机需要 50Hz PWM：

频率 = F_clk / ((PSC + 1) × (ARR + 1))
50Hz = 72MHz / ((PSC + 1) × (ARR + 1))

配置方案：
    PSC = 1439   → 分频后 50kHz
    ARR = 999    → 计数 1000 次 = 50Hz？不对！

正确计算：
    目标：50Hz，周期 20ms
    72MHz / 50Hz = 1440000
    
    方案：PSC = 71, ARR = 19999
    72MHz / 72 = 1MHz
    1MHz / 20000 = 50Hz
    
    或者：PSC = 1439, ARR = 999
    72MHz / 1440 = 50kHz
    50kHz / 1000 = 50Hz

推荐：PSC = 71, ARR = 19999（分辨率更高）
    分辨率 = 20000，可以精确到 0.009ms
```

### 3.2 角度与脉宽对应

```
角度 → 脉宽（CCR）：

角度   脉宽(ms)   CCR值
─────┼─────────┼────────
  0° │  0.5    │   500
 30° │  0.833  │   833
 60° │  1.167  │  1167
 90° │  1.5    │  1500
120° │  1.833  │  1833
150° │  2.167  │  2167
180° │  2.5    │  2500

公式：
    CCR = 500 + angle × 11.11
    
    其中 11.11 ≈ (2500 - 500) / 180
```

---

## 四、电路连接

### 4.1 硬件清单

| 组件 | 数量 | 说明 |
|------|------|------|
| STM32F103C8T6 最小系统板 | 1 | - |
| SG90 舵机 | 1-2 | 或其他 180° 舵机 |
| 外部电源 | 1 | 5V（舵机功耗较大） |
| 杜邦线 | 若干 | 连接电路 |

### 4.2 舵机引脚说明

| 舵机线色 | 功能 | 连接 |
|----------|------|------|
| 棕色 | GND | GND |
| 红色 | VCC | 5V（外部电源） |
| 橙色 | 信号 | PWM 输出引脚 |

### 4.3 电路图

```
        STM32F103C8T6                舵机 SG90
       ┌─────────────┐              ┌─────────┐
       │             │              │         │
       │    PA0 ─────┼──────────────┤ 信号(橙)│
       │  (TIM2_CH1) │              │         │
       │             │              │         │
       │    GND ─────┼──────────────┤ GND(棕) │
       │             │              │         │
       └─────────────┘              │         │
                                    │         │
            外部5V电源 ─────────────┤ VCC(红) │
                                    │         │
                                    └─────────┘

重要提示：
    1. 舵机工作时电流较大（可达 500mA）
    2. 不要用 STM32 的 5V 引脚直接供电
    3. 建议使用独立的外部 5V 电源
    4. GND 必须共地
```

---

## 五、完整代码实现

### 5.1 实验一：舵机角度控制

```c
#include "stm32f10x.h"

// ==================== PWM 初始化（50Hz）====================
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

    // 时基配置（50Hz PWM）
    // 72MHz / 72 = 1MHz, 1MHz / 20000 = 50Hz
    TIM_TimeBaseStructure.TIM_Period = 19999;      // ARR
    TIM_TimeBaseStructure.TIM_Prescaler = 71;      // PSC
    TIM_TimeBaseStructure.TIM_ClockDivision = 0;
    TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInit(TIM2, &TIM_TimeBaseStructure);

    // PWM 配置
    TIM_OCInitStructure.TIM_OCMode = TIM_OCMode_PWM1;
    TIM_OCInitStructure.TIM_OutputState = TIM_OutputState_Enable;
    TIM_OCInitStructure.TIM_Pulse = 1500;          // 初始 90°
    TIM_OCInitStructure.TIM_OCPolarity = TIM_OCPolarity_High;
    TIM_OC1Init(TIM2, &TIM_OCInitStructure);

    TIM_OC1PreloadConfig(TIM2, TIM_OCPreload_Enable);
    TIM_ARRPreloadConfig(TIM2, ENABLE);
    TIM_Cmd(TIM2, ENABLE);
}

// ==================== 设置舵机角度 ====================
void Servo_SetAngle(uint8_t angle)
{
    uint16_t ccr;
    if (angle > 180) angle = 180;

    // CCR = 500 + angle * 2000 / 180
    ccr = 500 + (uint32_t)angle * 2000 / 180;
    TIM_SetCompare1(TIM2, ccr);
}

// ==================== 延时函数 ====================
void delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 8000; j++);
}

// ==================== 主函数 ====================
int main(void)
{
    PWM_Init();

    // 初始位置：90°
    Servo_SetAngle(90);
    delay_ms(1000);

    while (1)
    {
        // 从 0° 扫描到 180°
        for (uint8_t angle = 0; angle <= 180; angle += 5)
        {
            Servo_SetAngle(angle);
            delay_ms(50);
        }

        // 从 180° 扫描回 0°
        for (uint8_t angle = 180; angle >= 0; angle -= 5)
        {
            Servo_SetAngle(angle);
            delay_ms(50);
        }
    }
}
```

### 5.2 实验二：按键控制舵机角度

```c
#include "stm32f10x.h"

uint8_t current_angle = 90;

void PWM_Init(void)
{
    // ... 同上 ...
}

void Servo_SetAngle(uint8_t angle)
{
    if (angle > 180) angle = 180;
    TIM_SetCompare1(TIM2, 500 + (uint32_t)angle * 2000 / 180);
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

    if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_1) == 0 && key_state == 0)
    {
        key_state = 1;
        return 1;  // KEY1：增加角度
    }
    else if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_2) == 0 && key_state == 0)
    {
        key_state = 1;
        return 2;  // KEY2：减少角度
    }
    else if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_1) != 0 &&
             GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_2) != 0)
    {
        key_state = 0;
    }
    return 0;
}

void delay_ms(uint32_t ms)
{
    uint32_t i, j;
    for (i = 0; i < ms; i++)
        for (j = 0; j < 8000; j++);
}

int main(void)
{
    uint8_t key;

    PWM_Init();
    GPIO_Config();

    Servo_SetAngle(current_angle);

    while (1)
    {
        key = Key_Scan();

        if (key == 1)  // 增加角度
        {
            if (current_angle < 180) current_angle += 10;
            Servo_SetAngle(current_angle);
        }
        else if (key == 2)  // 减少角度
        {
            if (current_angle > 0) current_angle -= 10;
            Servo_SetAngle(current_angle);
        }

        delay_ms(50);
    }
}
```

### 5.3 实验三：舵机平滑运动

```c
#include "stm32f10x.h"

// ==================== 舵机平滑运动 ====================
void Servo_MoveTo(uint8_t target_angle, uint16_t speed)
{
    static uint8_t current_angle = 90;

    while (current_angle != target_angle)
    {
        if (current_angle < target_angle)
            current_angle++;
        else if (current_angle > target_angle)
            current_angle--;

        Servo_SetAngle(current_angle);
        delay_ms(speed);  // speed 控制运动速度
    }
}

int main(void)
{
    PWM_Init();

    while (1)
    {
        // 平滑运动到 0°，速度 10ms/度
        Servo_MoveTo(0, 10);
        delay_ms(1000);

        // 平滑运动到 180°，速度 20ms/度
        Servo_MoveTo(180, 20);
        delay_ms(1000);

        // 平滑运动到 90°，速度 5ms/度
        Servo_MoveTo(90, 5);
        delay_ms(1000);
    }
}
```

---

## 六、调试技巧

### 6.1 常见问题排查

| 问题 | 可能原因 | 解决方法 |
|------|----------|----------|
| 舵机抖动 | PWM 频率不对 | 确认为 50Hz |
| 舵机不转 | 电源不足 | 使用外部 5V 电源 |
| 角度不对 | CCR 计算错误 | 检查公式和 ARR 值 |
| 舵机发热 | 长时间堵转 | 避免超过机械限位 |

### 6.2 注意事项

```
1. 供电问题：
   - 舵机启动电流大，不要用 STM32 供电
   - 建议使用独立 5V 电源
   - GND 必须共地

2. 机械限位：
   - 不要强制转动超过限位
   - 测试时先小角度，确认正常后再大角度

3. PWM 频率：
   - 标准 50Hz
   - 有些舵机支持更高频率（如 100Hz）

4. 角度校准：
   - 不同舵机可能有差异
   - 实际测试调整 CCR 范围
```

---

## 七、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【舵机 PWM】50Hz，周期 20ms                         │
│  【脉宽范围】0.5ms(0°) ~ 2.5ms(180°)                │
│  【CCR 计算】CCR = 500 + angle × 11.11             │
│  【独立供电】舵机电流大，需要外部电源               │
│  【平滑运动】逐度变化 + 延时                        │
└─────────────────────────────────────────────────────┘
```

### 参数速查表

```
┌─────────────────────────────────────────────────────┐
│  PWM 参数                                          │
│  ────────────────────────────────────────          │
│  频率    │  50Hz                                   │
│  周期    │  20ms                                   │
│  PSC     │  71（72MHz → 1MHz）                    │
│  ARR     │  19999（1MHz → 50Hz）                  │
├─────────────────────────────────────────────────────┤
│  脉宽与角度                                        │
│  ────────────────────────────────────────          │
│  0°     │  0.5ms  │  CCR = 500                    │
│  90°    │  1.5ms  │  CCR = 1500                   │
│  180°   │  2.5ms  │  CCR = 2500                   │
└─────────────────────────────────────────────────────┘
```

### 记忆口诀

```
舵机控制用 50 赫，周期二十毫秒整。
五百到两千五百，对应零到一百八。
独立供电很重要，否则电机转不动。
```

---

## 八、拓展练习

1. **双舵机控制**：使用两个 PWM 通道控制两个舵机
2. **机械臂模拟**：多个舵机模拟简单机械臂
3. **循迹小车**：舵机控制方向 + 电机驱动
4. **舵机+超声波**：制作扫描式距离测量仪

---

*上一篇：Day 09 - PWM 呼吸灯*  
*下一篇：Day 11 - 输入捕获*