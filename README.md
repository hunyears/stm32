# STM32 嵌入式学习笔记

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Author](https://img.shields.io/badge/author-Strive-orange.svg)](https://github.com/Strive)
[![Platform](https://img.shields.io/badge/platform-STM32F103-green.svg)]()
[![Language](https://img.shields.io/badge/language-C-blueviolet.svg)]()

> 🚀 从零开始学习 STM32 单片机开发的完整记录（30天学习计划）

## 📝 项目简介

本项目是作者 **Strive** 从 Java 转型嵌入式开发的完整学习记录。通过 30 天系统化的学习计划，从 GPIO 基础到综合项目实战，全面掌握 STM32F103 开发技能。

**本项目已开源，欢迎各位学习者、开发者指正和交流！** ⭐ Star 支持一下~

### ✨ 特色

- 📚 **系统化学习**：30 天循序渐进，从入门到项目实战
- 💡 **原理图解**：每个知识点配有详细图解和时序分析
- 🔧 **完整代码**：每个实验包含可运行的完整代码示例
- 🎯 **项目驱动**：最终完成智能温控系统综合项目
- 📖 **中文文档**：全中文教程，适合国内开发者

---

## 📚 学习进度总览

### 第一周：GPIO 与中断基础（Day 01-07）

| 日期 | 主题 | 笔记 | 状态 |
|------|------|------|------|
| 2026-07-12 | Day 01 - GPIO 点灯 | [📖 笔记](docs/day01-led/gpio-output.md) | ✅ 已完成 |
| 2026-07-13 | Day 02 - 跑马灯 | [📖 笔记](docs/day02-running-led/running-led.md) | ✅ 已完成 |
| 2026-07-14 | Day 03 - 蜂鸣器 | [📖 笔记](docs/day03-buzzer/buzzer.md) | ✅ 已完成 |
| 2026-07-15 | Day 04 - 按键输入与传感器 | [📖 笔记](docs/day04-input/key-sensor.md) | ✅ 已完成 |
| 2026-07-21 | Day 05 - 按键消抖与状态机 | [📖 笔记](docs/day05/key-debounce.md) | ✅ 已完成 |
| 2026-07-22 | Day 06 - 外部中断 EXTI | [📖 笔记](docs/day06/external-interrupt.md) | 🚀 进行中 |
| 2026-07-23 | Day 07 - 定时器中断 | [📖 笔记](docs/day07/timer-interrupt.md) | 🔜 待学习 |

### 第二周：定时器与 PWM（Day 08-14）

| 日期 | 主题 | 笔记 | 状态 |
|------|------|------|------|
| 2026-07-24 | Day 08 - PWM 输出基础 | [📖 笔记](docs/day08/pwm-basics.md) | 🔜 待学习 |
| 2026-07-25 | Day 09 - PWM 呼吸灯 | [📖 笔记](docs/day09/pwm-breathing-led.md) | 🔜 待学习 |
| 2026-07-26 | Day 10 - PWM 舵机控制 | [📖 笔记](docs/day10/pwm-servo.md) | 🔜 待学习 |
| 2026-07-27 | Day 11 - 输入捕获 | [📖 笔记](docs/day11/input-capture.md) | 🔜 待学习 |
| 2026-07-28 | Day 12 - USART 串口基础 | [📖 笔记](docs/day12/usart-basics.md) | 🔜 待学习 |
| 2026-07-29 | Day 13 - USART 中断与 DMA | [📖 笔记](docs/day13/usart-interrupt-dma.md) | 🔜 待学习 |
| 2026-07-30 | Day 14 - 串口应用实战 | [📖 笔记](docs/day14/usart-practice.md) | 🔜 待学习 |

### 第三周：I2C、SPI 与 ADC（Day 15-21）

| 日期 | 主题 | 笔记 | 状态 |
|------|------|------|------|
| 2026-07-31 | Day 15 - I2C 协议基础 | [📖 笔记](docs/day15/i2c-basics.md) | 🔜 待学习 |
| 2026-08-01 | Day 16 - I2C OLED 显示屏 | [📖 笔记](docs/day16/i2c-oled.md) | 🔜 待学习 |
| 2026-08-02 | Day 17 - I2C 温湿度传感器 | [📖 笔记](docs/day17/i2c-aht10.md) | 🔜 待学习 |
| 2026-08-03 | Day 18 - SPI 协议基础 | [📖 笔记](docs/day18/spi-basics.md) | 🔜 待学习 |
| 2026-08-04 | Day 19 - SPI Flash 存储 | [📖 笔记](docs/day19/spi-flash.md) | 🔜 待学习 |
| 2026-08-05 | Day 20 - ADC 多通道采集 | [📖 笔记](docs/day20/adc-multi-channel.md) | 🔜 待学习 |
| 2026-08-06 | Day 21 - DAC 模拟输出 | [📖 笔记](docs/day21/dac-output.md) | 🔜 待学习 |

### 第四周：综合项目实战（Day 22-30）

| 日期 | 主题 | 笔记 | 状态 |
|------|------|------|------|
| 2026-08-07 | Day 22 - RTC 实时时钟 | [📖 笔记](docs/day22/rtc-clock.md) | 🔜 待学习 |
| 2026-08-08 | Day 23 - 看门狗 | [📖 笔记](docs/day23/watchdog.md) | 🔜 待学习 |
| 2026-08-09 | Day 24 - 低功耗模式 | [📖 笔记](docs/day24/low-power.md) | 🔜 待学习 |
| 2026-08-10 | Day 25 - NVIC 中断管理进阶 | [📖 笔记](docs/day25/nvic-advanced.md) | 🔜 待学习 |
| 2026-08-11 | Day 26 - 项目设计：智能温控系统 | [📖 笔记](docs/day26/project-design.md) | 🔜 待学习 |
| 2026-08-12 | Day 27 - 项目实现（一）：传感器模块 | [📖 笔记](docs/day27/project-sensor.md) | 🔜 待学习 |
| 2026-08-13 | Day 28 - 项目实现（二）：显示与通信 | [📖 笔记](docs/day28/project-display.md) | 🔜 待学习 |
| 2026-08-14 | Day 29 - 项目实现（三）：控制逻辑 | [📖 笔记](docs/day29/project-control.md) | 🔜 待学习 |
| 2026-08-15 | Day 30 - 总结与进阶方向 | [📖 笔记](docs/day30/summary.md) | 🔜 待学习 |

---

## 🎯 学习目标

### 已掌握
- [x] 掌握 GPIO 基本输出（点灯）
- [x] 掌握多 GPIO 控制（跑马灯）
- [x] 掌握蜂鸣器控制
- [x] 掌握按键输入与消抖
- [x] 理解上拉电阻与下拉电阻原理
- [x] 掌握温度传感器（DS18B20）原理
- [x] 掌握光敏传感器与 ADC 采集
- [x] 掌握按键扫描与状态机消抖

### 待学习
- [ ] 掌握外部中断 EXTI
- [ ] 掌握定时器基本用法
- [ ] 掌握 PWM 输出与呼吸灯
- [ ] 掌握舵机控制
- [ ] 掌握输入捕获测量脉宽
- [ ] 掌握 USART 串口通信
- [ ] 掌握串口中断与 DMA
- [ ] 掌握 I2C 总线通信
- [ ] 掌握 OLED 显示屏驱动
- [ ] 掌握温湿度传感器（AHT10）
- [ ] 掌握 SPI 总线通信
- [ ] 掌握 Flash 存储器读写
- [ ] 掌握 ADC 多通道采集
- [ ] 掌握 DAC 模拟输出
- [ ] 掌握 RTC 实时时钟
- [ ] 掌握看门狗机制
- [ ] 掌握低功耗模式
- [ ] 完成综合项目实战

---

## 📂 目录结构

```
stm32/
├── README.md                    # 项目总览（你正在看这个）
├── docs/                        # 学习笔记
│   ├── day01-led/               # Day01: GPIO 点灯
│   ├── day02-running-led/       # Day02: 跑马灯
│   ├── day03-buzzer/            # Day03: 蜂鸣器
│   ├── day04-input/             # Day04: 按键输入与传感器
│   ├── day05/                   # Day05: 按键消抖与状态机
│   ├── day06/                   # Day06: 外部中断 EXTI
│   ├── day07/                   # Day07: 定时器中断
│   ├── day08/                   # Day08: PWM 输出基础
│   ├── day09/                   # Day09: PWM 呼吸灯
│   ├── day10/                   # Day10: PWM 舵机控制
│   ├── day11/                   # Day11: 输入捕获
│   ├── day12/                   # Day12: USART 串口基础
│   ├── day13/                   # Day13: USART 中断与 DMA
│   ├── day14/                   # Day14: 串口应用实战
│   ├── day15/                   # Day15: I2C 协议基础
│   ├── day16/                   # Day16: I2C OLED 显示屏
│   ├── day17/                   # Day17: I2C 温湿度传感器
│   ├── day18/                   # Day18: SPI 协议基础
│   ├── day19/                   # Day19: SPI Flash 存储
│   ├── day20/                   # Day20: ADC 多通道采集
│   ├── day21/                   # Day21: DAC 模拟输出
│   ├── day22/                   # Day22: RTC 实时时钟
│   ├── day23/                   # Day23: 看门狗
│   ├── day24/                   # Day24: 低功耗模式
│   ├── day25/                   # Day25: NVIC 中断管理进阶
│   ├── day26/                   # Day26: 项目设计
│   ├── day27/                   # Day27: 项目实现（一）
│   ├── day28/                   # Day28: 项目实现（二）
│   ├── day29/                   # Day29: 项目实现（三）
│   ├── day30/                   # Day30: 总结与进阶
│   ├── images/                  # 笔记图片（按天分类）
│   └── code-snippets/           # 常用代码片段速查
└── resources/                   # 参考资料 & 外链
```

---

## 🔧 开发环境

| 项目 | 说明 |
|------|------|
| **芯片** | STM32F103C8T6 |
| **开发板** | STM32F103C8T6 最小系统板 |
| **IDE** | Keil MDK / STM32CubeIDE |
| **烧录器** | ST-Link V2 |
| **固件库** | STM32 标准外设库 |
| **调试工具** | USB-TTL（CH340）、逻辑分析仪 |

---

## 🚀 快速开始

### 1. 克隆项目

```bash
git clone https://github.com/Strive/stm32-learning.git
```

### 2. 学习路径

建议按顺序学习，每天完成一个主题：

1. **第一周**：GPIO 与中断基础
2. **第二周**：定时器与 PWM
3. **第三周**：通信协议（I2C/SPI）与模拟外设
4. **第四周**：系统功能与项目实战

### 3. 硬件准备

- STM32F103C8T6 最小系统板
- ST-Link V2 烧录器
- USB-TTL 模块（串口调试）
- 常用元器件（LED、按键、蜂鸣器、传感器等）

---

## 🤝 贡献指南

欢迎各位学习者提出宝贵意见！

- 🐛 发现错误？[提交 Issue](../../issues)
- 💡 有建议？[参与讨论](../../discussions)
- 🔧 想贡献代码？欢迎提交 Pull Request

### 如何贡献

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 提交 Pull Request

---

## 📖 知识点速查

### GPIO 相关
- `RCC_APB2PeriphClockCmd()` - 开启时钟
- `GPIO_Init()` - 初始化 GPIO
- `GPIO_SetBits()` / `GPIO_ResetBits()` - 输出高低电平
- `GPIO_ReadInputDataBit()` - 读取输入电平

### 中断相关
- `NVIC_PriorityGroupConfig()` - 设置优先级分组
- `NVIC_Init()` - 初始化中断通道
- `EXTI_Init()` - 初始化外部中断
- `EXTI_GetITStatus()` - 获取中断状态
- `EXTI_ClearITPendingBit()` - 清除中断标志

### 定时器相关
- `TIM_TimeBaseInit()` - 定时器时基初始化
- `TIM_OCInit()` - 输出比较初始化
- `TIM_Cmd()` - 使能定时器
- `TIM_ITConfig()` - 中断配置

### 通信相关
- `USART_Init()` - 串口初始化
- `USART_SendData()` - 发送数据
- `I2C_Init()` - I2C 初始化
- `SPI_Init()` - SPI 初始化

---

## 📜 License

本项目采用 [MIT License](LICENSE) 开源协议。

---

## 🙏 致谢

感谢以下资源和社区的帮助：

- [STM32 官方文档](https://www.st.com/zh-CN/microcontrollers-microprocessors/stm32-32-bit-arm-cortex-mcus.html)
- [正点原子](http://www.openedv.com/)
- [野火电子](https://doc.embedfire.com/)
- [STM32 中文论坛](https://www.stmcu.org.cn/)

---

<div align="center">

**⭐ 如果这个项目对你有帮助，请给一个 Star ⭐**

*学习始于 2026-07-12，坚持每天进步一点点 💪*

Made with ❤️ by **Strive**

</div>