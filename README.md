# STM32 嵌入式学习笔记

> 从零开始学习 STM32 单片机开发的完整记录

## 📚 学习进度

| 日期 | 主题 | 笔记 | 状态 |
|------|------|------|------|
| 2026-07-12 | Day 01 - GPIO 点灯 | [📖 笔记](docs/day01-led/README.md) | ✅ 已完成 |
| 2026-07-13 | Day 02 - 跑马灯 | [📖 笔记](docs/day02-running-led/README.md) | ✅ 已完成 |
| 2026-07-14 | Day 03 - 蜂鸣器 | [📖 笔记](docs/day03-buzzer/README.md) | ✅ 已完成 |
| 2026-07-15 | Day 04 - 按键输入与传感器 | [📖 笔记](docs/day04-input/README.md) | ✅ 已完成 |
| - | Day 05 - EXTI 中断 | - | 🔜 待学习 |
| - | Day 06 - 定时器 | - | 🔜 待学习 |
| - | Day 07 - PWM 输出 | - | 🔜 待学习 |
| - | Day 08 - USART 串口 | - | 🔜 待学习 |
| - | Day 09 - I2C 通信 | - | 🔜 待学习 |
| - | Day 10 - SPI 通信 | - | 🔜 待学习 |

## 📂 目录结构

```
stm32/
├── README.md                    # 项目总览（你正在看这个）
├── docs/                        # 学习笔记
│   ├── day01-led/               # Day01: GPIO 点灯
│   │   └── README.md
│   ├── day02-running-led/       # Day02: 跑马灯
│   │   └── README.md
│   ├── day03-buzzer/            # Day03: 蜂鸣器
│   │   └── README.md
│   ├── day04-input/             # Day04: 按键输入与传感器
│   │   └── README.md
│   ├── images/                  # 笔记图片（按天分类）
│   │   ├── day01-led/
│   │   ├── day02-running-led/
│   │   ├── day03-buzzer/
│   │   └── day04-input/
│   └── code-snippets/           # 常用代码片段速查
│       └── README.md
└── resources/                   # 参考资料 & 外链
    └── README.md
```

## 🎯 学习目标

- [x] 掌握 GPIO 基本输出（点灯）
- [x] 掌握多 GPIO 控制（跑马灯）
- [x] 掌握蜂鸣器控制
- [x] 掌握按键输入与消抖
- [x] 理解上拉电阻与下拉电阻原理
- [x] 掌握温度传感器（DS18B20）原理
- [x] 掌握光敏传感器与 ADC 采集
- [ ] 掌握外部中断 EXTI
- [ ] 掌握定时器基本用法
- [ ] 掌握 PWM 输出
- [ ] 掌握 USART 串口通信
- [ ] 掌握 I2C 总线通信
- [ ] 掌握 SPI 总线通信

## 🔧 开发环境

- **芯片**: STM32F103
- **开发板**: STM32F103C8T6 最小系统板
- **IDE**: Keil MDK / STM32CubeIDE
- **烧录器**: ST-Link
- **库**: 标准库

---

*学习始于 2026-07-12，坚持每天进步一点点 💪*