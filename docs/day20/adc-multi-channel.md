# Day 20 - ADC 多通道采集学习笔记

> 学习日期：2026-08-05

---

## 一、核心知识点

### 1.1 ADC 基础回顾

#### ADC（模数转换器）

```
STM32F103 ADC 特点：
    - 分辨率：12 位（0~4095）
    - 通道数：16 个外部通道 + 2 个内部
    - 转换时间：最快 1μs
    - 参考电压：3.3V

电压转换：
    V = ADC值 × 3.3 / 4095
```

### 1.2 多通道采集模式

```
单次转换模式：
    每次只转换一个通道，需手动切换

连续转换模式：
    自动连续转换指定通道

扫描模式（多通道）：
    自动扫描多个通道，依次转换
    配合 DMA 实现高效数据搬运

DMA 模式：
    ADC 转换完成自动写入内存
    无需 CPU 参与
```

---

## 二、常用库函数

```c
// ADC 初始化
void ADC_Init(ADC_TypeDef* ADCx, ADC_InitTypeDef* ADC_InitStruct);

// 配置通道顺序
void ADC_RegularChannelConfig(ADC_TypeDef* ADCx, uint8_t ADC_Channel,
                               uint8_t Rank, uint8_t ADC_SampleTime);

// 使能 DMA
void ADC_DMACmd(ADC_TypeDef* ADCx, FunctionalState NewState);

// 启动转换
void ADC_SoftwareStartConvCmd(ADC_TypeDef* ADCx, FunctionalState NewState);
```

---

## 三、完整代码实现

### 3.1 ADC 多通道 DMA 配置

```c
#define ADC_CHANNELS 3
uint16_t adc_value[ADC_CHANNELS];

void ADC_MultiChannel_Init(void)
{
    GPIO_InitTypeDef GPIO_InitStructure;
    ADC_InitTypeDef ADC_InitStructure;
    DMA_InitTypeDef DMA_InitStructure;

    // 开启时钟
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_ADC1, ENABLE);
    RCC_AHBPeriphClockCmd(RCC_AHBPeriph_DMA1, ENABLE);

    // GPIO 配置（PA0, PA1, PA2）
    GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0 | GPIO_Pin_1 | GPIO_Pin_2;
    GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AIN;
    GPIO_Init(GPIOA, &GPIO_InitStructure);

    // DMA 配置
    DMA_DeInit(DMA1_Channel1);
    DMA_InitStructure.DMA_PeripheralBaseAddr = (uint32_t)&ADC1->DR;
    DMA_InitStructure.DMA_MemoryBaseAddr = (uint32_t)adc_value;
    DMA_InitStructure.DMA_DIR = DMA_DIR_PeripheralSRC;
    DMA_InitStructure.DMA_BufferSize = ADC_CHANNELS;
    DMA_InitStructure.DMA_PeripheralInc = DMA_PeripheralInc_Disable;
    DMA_InitStructure.DMA_MemoryInc = DMA_MemoryInc_Enable;
    DMA_InitStructure.DMA_PeripheralDataSize = DMA_PeripheralDataSize_HalfWord;
    DMA_InitStructure.DMA_MemoryDataSize = DMA_MemoryDataSize_HalfWord;
    DMA_InitStructure.DMA_Mode = DMA_Mode_Circular;
    DMA_InitStructure.DMA_Priority = DMA_Priority_High;
    DMA_InitStructure.DMA_M2M = DMA_M2M_Disable;
    DMA_Init(DMA1_Channel1, &DMA_InitStructure);
    DMA_Cmd(DMA1_Channel1, ENABLE);

    // ADC 配置
    ADC_InitStructure.ADC_Mode = ADC_Mode_Independent;
    ADC_InitStructure.ADC_ScanConvMode = ENABLE;  // 扫描模式
    ADC_InitStructure.ADC_ContinuousConvMode = ENABLE;
    ADC_InitStructure.ADC_ExternalTrigConv = ADC_ExternalTrigConv_None;
    ADC_InitStructure.ADC_DataAlign = ADC_DataAlign_Right;
    ADC_InitStructure.ADC_NbrOfChannel = ADC_CHANNELS;
    ADC_Init(ADC1, &ADC_InitStructure);

    // 配置通道
    ADC_RegularChannelConfig(ADC1, ADC_Channel_0, 1, ADC_SampleTime_55Cycles5);
    ADC_RegularChannelConfig(ADC1, ADC_Channel_1, 2, ADC_SampleTime_55Cycles5);
    ADC_RegularChannelConfig(ADC1, ADC_Channel_2, 3, ADC_SampleTime_55Cycles5);

    // 使能 DMA
    ADC_DMACmd(ADC1, ENABLE);
    ADC_Cmd(ADC1, ENABLE);

    // 校准
    ADC_ResetCalibration(ADC1);
    while (ADC_GetResetCalibrationStatus(ADC1));
    ADC_StartCalibration(ADC1);
    while (ADC_GetCalibrationStatus(ADC1));

    ADC_SoftwareStartConvCmd(ADC1, ENABLE);
}
```

### 3.2 读取电压值

```c
float Get_Voltage(uint8_t channel)
{
    return (float)adc_value[channel] * 3.3f / 4095.0f;
}

int main(void)
{
    ADC_MultiChannel_Init();

    while (1)
    {
        float v0 = Get_Voltage(0);  // PA0 电压
        float v1 = Get_Voltage(1);  // PA1 电压
        float v2 = Get_Voltage(2);  // PA2 电压
    }
}
```

---

## 四、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【扫描模式】ADC_ScanConvMode = ENABLE             │
│  【通道配置】设置 Rank 确定转换顺序                │
│  【DMA 配合】循环模式自动更新数组                   │
│  【电压计算】V = ADC × 3.3 / 4095                   │
└─────────────────────────────────────────────────────┘
```

---

*上一篇：Day 19 - SPI Flash 存储*  
*下一篇：Day 21 - DAC 模拟输出*