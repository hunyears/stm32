# Day 21 - DAC 模拟输出学习笔记

> 学习日期：2026-08-06

---

## 一、核心知识点

### 1.1 什么是 DAC？

#### 定义

DAC（Digital to Analog Converter，数模转换器）将数字信号转换为模拟电压。

```
STM32F103 DAC 特点：
    - 分辨率：12 位
    - 通道：2 个（DAC1/PA4, DAC2/PA5）
    - 输出范围：0 ~ VREF（通常 3.3V）
    - 输出公式：V = DAC_Value × 3.3 / 4095

应用场景：
    - 波形生成（正弦波、三角波）
    - 音频输出
    - 电机控制参考电压
    - 模拟信号仿真
```

---

## 二、常用库函数

```c
// DAC 初始化
void DAC_Init(DAC_TypeDef* DACx, uint32_t DAC_Channel, DAC_InitTypeDef* DAC_InitStruct);

// 使能 DAC
void DAC_Cmd(DAC_TypeDef* DACx, uint32_t DAC_Channel, FunctionalState NewState);

// 设置输出值
void DAC_SetChannel1Data(DAC_TypeDef* DACx, uint16_t Data);
void DAC_SetChannel2Data(DAC_TypeDef* DACx, uint16_t Data);

// 触发转换
void DAC_SoftwareTriggerCmd(DAC_TypeDef* DACx, uint32_t DAC_Channel, FunctionalState NewState);
```

---

## 三、完整代码实现

### 3.1 DAC 基本输出

```c
void DAC1_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    DAC_InitTypeDef DAC_InitStructure;

    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_DAC, ENABLE);

    // PA4 配置为模拟输出
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_4;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // DAC 配置
    DAC_InitStructure.DAC_Trigger = DAC_Trigger_None;
    DAC_InitStructure.DAC_WaveGeneration = DAC_WaveGeneration_None;
    DAC_InitStructure.DAC_LFSRUnmask_TriangleAmplitude = DAC_LFSRUnmask_Bit0;
    DAC_InitStructure.DAC_OutputBuffer = DAC_OutputBuffer_Enable;
    DAC_Init(DAC, DAC_Channel_1, &DAC_InitStructure);

    DAC_Cmd(DAC, DAC_Channel_1, ENABLE);
}

void DAC_SetVoltage(float voltage)
{
    uint16_t value = (uint16_t)(voltage * 4095 / 3.3);
    DAC_SetChannel1Data(DAC, value);
}
```

### 3.2 正弦波生成

```c
// 正弦波查找表（100点）
const uint16_t sine_table[100] = {
    2048, 2176, 2304, 2430, 2554, 2675, 2793, 2906, 3015, 3118,
    // ... 完整正弦表
};

void DAC_SineWave(void)
{
    uint8_t i = 0;
    while (1)
    {
        DAC_SetChannel1Data(DAC, sine_table[i]);
        i = (i + 1) % 100;
        delay_us(100);  // 控制频率
    }
}
```

### 3.3 三角波生成

```c
void DAC_TriangleWave(void)
{
    uint16_t value = 0;
    uint8_t direction = 0;

    while (1)
    {
        if (direction == 0)
        {
            DAC_SetChannel1Data(DAC, value++);
            if (value >= 4095) direction = 1;
        }
        else
        {
            DAC_SetChannel1Data(DAC, value--);
            if (value == 0) direction = 0;
        }
        delay_us(50);
    }
}
```

---

## 四、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【DAC 引脚】PA4（DAC1）、PA5（DAC2）               │
│  【输出范围】0 ~ 3.3V                               │
│  【分辨率】12 位（0~4095）                          │
│  【波形生成】查找表 + 定时更新                      │
└─────────────────────────────────────────────────────┘
```

---

*上一篇：Day 20 - ADC 多通道采集*  
*下一篇：Day 22 - RTC 实时时钟*