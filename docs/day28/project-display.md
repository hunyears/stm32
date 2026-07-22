# Day 28 - 项目实现（二）：显示与通信学习笔记

> 学习日期：2026-08-13

---

## 一、显示模块实现

### 1.1 OLED 界面设计

```c
void Display_Main(void)
{
    char buf[32];

    // 清屏
    OLED_Clear();

    // 标题
    OLED_ShowString(0, 0, "Smart Temp Control");

    // 当前温度
    sprintf(buf, "Temp: %.1f C", current_temp);
    OLED_ShowString(0, 2, buf);

    // 目标温度
    sprintf(buf, "Target: %.1f C", target_temp);
    OLED_ShowString(0, 4, buf);

    // 状态
    switch (system_state)
    {
        case STATE_IDLE:    OLED_ShowString(0, 6, "State: IDLE"); break;
        case STATE_HEATING: OLED_ShowString(0, 6, "State: HEATING"); break;
        case STATE_COOLING: OLED_ShowString(0, 6, "State: COOLING"); break;
        case STATE_ALARM:   OLED_ShowString(0, 6, "State: ALARM!"); break;
    }
}
```

---

## 二、通信模块实现

### 2.1 数据上报格式

```c
void Report_Data(void)
{
    printf("{\"temp\":%.1f,\"humi\":%.1f,\"state\":%d,\"time\":%ld}\r\n",
           current_temp, current_humi, system_state, RTC_GetCounter());
}
```

### 2.2 串口命令解析

```c
void Parse_Command(char *cmd)
{
    if (strncmp(cmd, "SET:", 4) == 0)
    {
        target_temp = atof(cmd + 4);
        printf("Target set to %.1f\r\n", target_temp);
    }
    else if (strcmp(cmd, "READ") == 0)
    {
        Report_Data();
    }
    else if (strcmp(cmd, "STATUS") == 0)
    {
        printf("System state: %d\r\n", system_state);
    }
}
```

---

## 三、总结

### 核心要点

```
┌─────────────────────────────────────────────────────┐
│  【界面设计】清晰显示关键信息                       │
│  【数据上报】JSON 格式便于解析                      │
│  【命令解析】简单文本协议易于调试                   │
└─────────────────────────────────────────────────────┘
```

---

*上一篇：Day 27 - 传感器模块*  
*下一篇：Day 29 - 项目实现（三）：控制逻辑*