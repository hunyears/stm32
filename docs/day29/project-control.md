# Day 29 - 项目实现（三）：控制逻辑学习笔记

> 学习日期：2026-08-14

---

## 一、PID 控制基础

### 1.1 PID 原理

```
PID = 比例(P) + 积分(I) + 微分(D)

输出 = Kp × e + Ki × ∫e + Kd × de/dt

其中：
    e = 目标值 - 当前值（误差）
    Kp = 比例系数
    Ki = 积分系数
    Kd = 微分系数
```

### 1.2 简化实现

```c
typedef struct {
    float Kp;
    float Ki;
    float Kd;
    float target;
    float integral;
    float last_error;
} PID_Controller;

float PID_Calculate(PID_Controller *pid, float current)
{
    float error = pid->target - current;
    pid->integral += error;

    float output = pid->Kp * error +
                   pid->Ki * pid->integral +
                   pid->Kd * (error - pid->last_error);

    pid->last_error = error;

    // 输出限幅
    if (output > 100) output = 100;
    if (output < 0) output = 0;

    return output;
}
```

---

## 二、继电器控制

```c
#define RELAY_HEAT_PIN  GPIO_Pin_8
#define RELAY_COOL_PIN  GPIO_Pin_9

void Relay_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);

    GPIO_InitStructure.GPIO_Pin = RELAY_HEAT_PIN | RELAY_COOL_PIN;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;
    GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
    GPIO_Init(GPIOB, &GPIO_InitStructure);

    GPIO_SetBits(GPIOB, RELAY_HEAT_PIN | RELAY_COOL_PIN);  // 断开
}

void Relay_On(uint16_t pin)
{
    GPIO_ResetBits(GPIOB, pin);  // 低电平触发
}

void Relay_Off(uint16_t pin)
{
    GPIO_SetBits(GPIOB, pin);
}
```

---

## 三、完整状态机

```c
void Control_Process(void)
{
    float temp = Get_Temperature();
    float output = PID_Calculate(&pid, temp);

    if (output > 50)
    {
        Relay_On(RELAY_HEAT_PIN);
        Relay_Off(RELAY_COOL_PIN);
        system_state = STATE_HEATING;
    }
    else if (output < -50)
    {
        Relay_Off(RELAY_HEAT_PIN);
        Relay_On(RELAY_COOL_PIN);
        system_state = STATE_COOLING;
    }
    else
    {
        Relay_Off(RELAY_HEAT_PIN);
        Relay_Off(RELAY_COOL_PIN);
        system_state = STATE_IDLE;
    }

    // 报警检测
    if (temp > target_temp + 10.0f || temp < target_temp - 10.0f)
    {
        system_state = STATE_ALARM;
        Buzzer_On();
    }
}
```

---

## 四、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【PID 控制】比例+积分+微分，精确控制               │
│  【状态机】清晰的状态转换逻辑                       │
│  【安全保护】温度超限自动报警                       │
└─────────────────────────────────────────────────────┘
```

---

*上一篇：Day 28 - 显示与通信*  
*下一篇：Day 30 - 总结与进阶方向*