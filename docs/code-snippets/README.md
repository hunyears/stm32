# 常用代码片段速查

> 学习过程中积累的常用代码片段，方便快速查阅和复用

## GPIO 输出配置

```c
// 使能 GPIO 时钟
RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOx, ENABLE);

// 配置 GPIO 为推挽输出
GPIO_InitTypeDef GPIO_InitStructure;
GPIO_InitStructure.GPIO_Pin = GPIO_Pin_x;
GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;  // 推挽输出
GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
GPIO_Init(GPIOx, &GPIO_InitStructure);

// 控制输出
GPIO_SetBits(GPIOx, GPIO_Pin_x);    // 输出高电平
GPIO_ResetBits(GPIOx, GPIO_Pin_x);  // 输出低电平
```

## 延时函数

```c
// 简单软件延时（不精确）
void Delay(uint32_t count) {
    while(count--);
}

// 使用 SysTick 精确延时
void Delay_ms(uint32_t ms) {
    while(ms--) {
        SysTick_Config(72000);  // 72MHz → 1ms
        while(!((SysTick->CTRL) & (1<<16)));
        SysTick->CTRL &= ~SysTick_CTRL_ENABLE_Msk;
    }
}
```

## 位操作技巧

```c
// 置位（设为1）
GPIOx->ODR |= (1 << pin);

// 清位（设为0）
GPIOx->ODR &= ~(1 << pin);

// 翻转
GPIOx->ODR ^= (1 << pin);

// 判断某位
if (GPIOx->IDR & (1 << pin)) { /* 高电平 */ }
```

---

*持续更新中...*