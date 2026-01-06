---
title: "嵌入式的初学问题"
date: 2026-01-06T19:56:00+08:00
draft: false
tags: ["车灯", "心情一般"]
categories: ["Think"]
---

# 前言

我转行进入软件已经半年多了，到目前位置只做过三个项目，每个项目都有一些值得去说的点和学习经验，好记性不如烂笔头，记下来常常阅读才有成效。

首先公司采用的软件架构是 **Time-Triggered Cooperative Architecture（时间触发的协作式架构）**，每 N ms 就执行一次任务，优点是：

- 没有抢占；
- 没有复杂同步；
- 没有死锁风险。

---

# 最近遇到的问题

## 1. 电源滤波问题

写电源管理模块的时候，采用平均平滑滤波的算法，也就是采用十次采样值平均，然后得到一个平滑的值，这样做的好处是留有反应时间，也不会被突然采样到的一个 max 或者 min 值使得灯突然灭掉一下。

测试的时候发现，这样做的话，如果开局就是过压，灯会亮一下然后灭掉，因为我初始十个值给的是 9V，所以一开始会进入大约 100ms 的常压过程。

于是我修改，但是修改后发现启动延时时间不够了，因为我设定了一个新的电源状态，就是 `detected_not_OK`，这样的话，如果是常压点灯，反而会过一会才会亮，而启动延时大于 30ms，不符合需求。

最终还是让它闪一下了。

---

## 2. 休眠失败 reset

这个是同事的问题让我自查的，我也加了相关模块。简单来说，就是 MCU 不一定能休眠成功，如果休眠失败，代码已经把部分外设关掉了，之后醒来的措施存在失灵的情况，导致醒来失败。

这个时候得搞清楚休眠具体步骤，一般休眠就是拉掉 DCDC 和 LDO，分为两种情况：

- MCU 断电
- MCU 有专门的休眠模式，还在低功耗运行

前者可以加一个计数的 counter，MCU 断电则 counter 停止计数，不会触发休眠 NG 措施。

而低功耗情况下，则需要检测 DCDC 和 LDO 状态、相关 Pin 脚电平，随时准备 reset。

---

## 3. flag 清零注意

在开发过程中，难免会遗漏 flag 未清零的状况，可以在草稿纸上把 flag 一个个列出来，然后检查下是否都在标志位用过后重置。

---

## 4. 输出和逻辑分开

在写故障诊断逻辑的时候，我曾经写过多次输出的垃圾代码，每当有条件判断的时候，我都会加比如 `DiagSet` 错误或者非错误，这样多次开关 GPIO，很容易出错而且软件运行时间很慢。

改成变量或者 enum 来存储故障，最后函数结尾一次输出即可。

在更大的项目中，同事的做法是把判断函数和执行函数分开的，中间由全局枚举承担状态切换功能。

---

# 5. 代码规范部分

## 5.1 命名规范

命名采用 **小写单词 + 下划线（snake_case）**

### ❌ 错误示例

```c
int TurnLampStatus;
void TurnLampFlowCtrl(void);
uint8_t LedChA;
```

问题：

- 使用了驼峰命名
- 大小写混用
- 不符合统一风格

### ✅ 正确示例

```c
int turn_lamp_status;
void turn_lamp_flow_ctrl(void);
uint8_t led_ch_a;
```

---

## 5.2 函数功能注释必须编写

### ❌ 错误示例（无注释）

```c
void turn_led_flow_ctrl(uint8_t mode)
{
    flow_state = mode;
}
```

### ✅ 正确示例（标准函数头注释）

```c
/**
 * @brief  Control turn lamp flow behavior
 *
 * @param  mode  Flow control mode (FLOW_ON / FLOW_OFF)
 *
 * @return None
 *
 * @note   This function is called periodically in 10ms task
 */
void turn_led_flow_ctrl(uint8_t mode)
{
    flow_state = mode;
}
```

---

## 5.3 #define 常量必须使用大写字母

### ❌ 错误示例

```c
#define flow_wait_time   8
#define max_led_ch       10
#define pwm_period_ms    40
```

问题：

- 宏名使用小写
- 难以区分宏与变量

### ✅ 正确示例

```c
#define FLOW_WAIT_TIME   8
#define MAX_LED_CH       10
#define PWM_PERIOD_MS   40
```

### ❌ 错误示例（宏被当变量用）

```c
#define turn_on   1
#define turn_off  0
```

### ✅ 正确示例（枚举更合适）

```c
typedef enum
{
    TURN_OFF = 0,
    TURN_ON  = 1
} turn_state_t;
```

---

## 5.4 头文件不允许多层、交叉包含

### ❌ 错误示例 1：多层包含

```c
/* a.h */
#include "b.h"

/* b.h */
#include "c.h"

/* c.h */
#include "a.h"   // ❌ 形成循环包含
```

问题：

- 循环依赖
- 编译顺序敏感
- 易产生隐性错误

---

### ❌ 错误示例 2：头文件包含实现头

```c
/* turn_lamp.h */
#include "tps929240.h"   // ❌ 应该放到 .c 中
```

问题：

- 头文件被“污染”
- 强耦合底层驱动

---

### ✅ 正确示例 1：头文件只声明，不依赖实现

```c
/* turn_lamp.h */
#ifndef TURN_LAMP_H
#define TURN_LAMP_H

#include <stdint.h>

/* forward declaration */
typedef struct tps929240_handle tps929240_handle_t;

void turn_lamp_init(void);
void turn_lamp_control(uint8_t state);

#endif /* TURN_LAMP_H */
```

---

### ✅ 正确示例 2：依赖放在 .c 文件中

```c
/* turn_lamp.c */
#include "turn_lamp.h"
#include "tps929240.h"   // ✅ 只在实现文件中包含
```

---

### ✅ 正确示例 3：防止重复包含（Include Guard）

```c
#ifndef TPS929240_H
#define TPS929240_H

void tps929240_init(void);

#endif /* TPS929240_H */
```

---

## 5.5 缩写必须有对应注释

```c
uint8_t tu_state;
uint16_t pwm_prd;
void tps_init(void);
```

问题：

- tu 含义不明
- prd 是 period 还是 product？
- tps 是芯片名还是功能？

### ✅ 正确示例：变量缩写 + 行内注释

```c
uint8_t  tu_state;   /* TU: Turn Unit state */
uint16_t pwm_prd;    /* PRD: PWM period in ms */
```

