# Day 27 - 项目实现（一）：传感器模块学习笔记

> 学习日期：2026-08-12

---

## 一、传感器模块设计

### 1.1 模块接口定义

```c
// 传感器数据结构
typedef struct {
    float temperature;    // 温度值
    float humidity;       // 湿度值
    uint8_t valid;        // 数据有效标志
    uint32_t timestamp;   // 时间戳
} Sensor_Data;

// 传感器接口
void Sensor_Init(void);
void Sensor_Read(Sensor_Data *data);
float Sensor_GetTemp(void);
float Sensor_GetHumi(void);
```

### 1.2 定时采集实现

```c
volatile uint8_t sensor_ready = 0;

void TIM2_IRQHandler(void)
{
    static uint16_t count = 0;

    if (TIM_GetITStatus(TIM2, TIM_IT_Update))
    {
        count++;
        if (count >= 1000)  // 1秒
        {
            count = 0;
            sensor_ready = 1;
        }
        TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
    }
}

void Sensor_Process(void)
{
    static Sensor_Data data;

    if (sensor_ready)
    {
        sensor_ready = 0;
        AHT10_Read(&data.temperature, &data.humidity);
        data.valid = 1;
        data.timestamp = RTC_GetCounter();
    }
}
```

### 1.3 滤波算法

```c
// 滑动平均滤波
#define FILTER_SIZE 5

float Filter_Temperature(float new_value)
{
    static float buffer[FILTER_SIZE];
    static uint8_t index = 0;
    float sum = 0;

    buffer[index] = new_value;
    index = (index + 1) % FILTER_SIZE;

    for (uint8_t i = 0; i < FILTER_SIZE; i++)
        sum += buffer[i];

    return sum / FILTER_SIZE;
}
```

---

## 二、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【定时采集】定时器中断触发，周期精确               │
│  【滤波算法】滑动平均消除抖动                       │
│  【数据结构】结构体封装便于管理                     │
└─────────────────────────────────────────────────────┘
```

---

*上一篇：Day 26 - 项目设计*  
*下一篇：Day 28 - 项目实现（二）：显示与通信*