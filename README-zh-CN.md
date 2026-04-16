# DShot-STM32

[English](README.md) | 简体中文

适用于 STM32 微控制器的 C++ DShot 协议库，使用 HAL 库和 DMA 传输。

## 简介

本库提供了基于 C++ 类的 DShot 协议实现，用于在 STM32 微控制器上控制无刷电调（ESC）。它使用 STM32 HAL 库配合 DMA 进行高效、非阻塞的电机控制。

### 特性

- **基于 DMA 的传输** - 非阻塞式电机控制
- **面向对象设计** - 代码简洁、可复用
- **FreeRTOS 支持** - 可选的非阻塞延时
- **多种 DShot 速率** - DShot150、DShot300、DShot600、DShot1200

- **已在 STM32F4 上测试** - 使用 BL_Heli32 DShot600 电调验证通过

## DShot 协议概述

DShot 是一种用于与电调通信的数字协议，相比传统 PWM 具有以下优势：
- **更高分辨率** - 2000 级（48-2047）
- **错误检测** - 内置 CRC 校验
- **无需校准** - 数字信号，油门值精确
- **双向通信** - 支持遥测（可选）

### 数据帧结构

![image](https://user-images.githubusercontent.com/48342925/105716366-0f1e2680-5f62-11eb-8d5c-651e15907a53.png)

| 位数 | 用途 | 范围 |
|:----:|:--------|:------|
| 11 位 | 油门 / 命令 | 0-47：命令，48-2047：油门 |
| 1 位 | 遥测请求 | 0：否，1：是 |
| 4 位 | CRC 校验和 | 0-15 |

### 位编码

![image](https://user-images.githubusercontent.com/48342925/105717632-b780ba80-5f63-11eb-9508-54b4be15900d.png)

| | 位 0 | 位 1 |
|:---:|:---:|:---:|
| 占空比 | 37.425% | 74.850% |

### 传输速率

| 模式 | 比特率 | 位时间 | 帧时间 | 更新率 |
|:---:|:---:|:---:|:---:|:---:|
| DShot150 | 150 Kbit/s | 6.67 µs | 106.7 µs | ~9.4 kHz |
| DShot300 | 300 Kbit/s | 3.33 µs | 53.3 µs | ~18.8 kHz |
| DShot600 | 600 Kbit/s | 1.67 µs | 26.7 µs | ~37.5 kHz |
| DShot1200 | 1200 Kbit/s | 0.83 µs | 13.3 µs | ~75 kHz |

### 信号示例

<img src="https://user-images.githubusercontent.com/48342925/122077706-ec1fe080-ce36-11eb-9f24-a70c651b41c5.jpg" width="50%">

- 油门值：11
- 遥测请求：0（否）
- 校验和：0111

### 解锁序列

电调上电后需要执行解锁序列才能响应油门命令：

1. **电调上电** - 等待初始化的3次提示音
2. **发送零油门** - 持续发送 `0` 约 3 秒
3. **等待确认音** - 一声高音表示解锁成功
4. **可以发送油门** - 现在可以发送油门值（48-2047）

<img src="https://user-images.githubusercontent.com/48342925/124644199-f466bb00-decc-11eb-8dc9-396d9b9706d7.png" width="70%">

## 硬件要求

- STM32 微控制器（F1、F4、F7、H7 系列）
- BLHeli_32 或兼容的 DShot 电调
- 无刷电机

## STM32CubeMX 配置

### 定时器配置

在所需通道上配置定时器为 PWM 生成模式：

![MX_TIM_Config](/example-images/mxconfig%20(1).png)
![MX_TIM_Config](/example-images/mxconfig%20(2).png)

**关键设置：**
- 通道：PWM 生成正常模式
- 预分频器：由库根据 DShot 速率自动设置
- 计数周期：20-1

### DMA 配置

为每个使用的定时器通道启用 DMA：

![MX_TIM_DMA_Config](/example-images/mxconfig%20(3).png)

**关键设置：**
- 方向：内存到外设
- 模式：普通
- 数据宽度：字（32 位）

## 使用方法

1. 将 `src/dshot.h` 和 `src/dshot.cpp` 复制到你的项目中
2. 在代码中包含 `dshot.h`
3. 按照上述说明配置 STM32CubeMX

### 基础示例

```cpp
#include "dshot.h"

// 为 TIM1 通道 1 创建 DShot 实例
DShot motor(&htim1, TIM_CHANNEL_1, DShot::DShotType::DSHOT600);

int main() {
    // 初始化电机
    if (!motor.begin()) {
        // 处理错误
    }
    
    // 解锁电调（上电后必须执行）
    motor.disarm();
    
    // 现在可以控制电机了
    while (1) {
        // ...
        motor.send(100); // 必须持续调用 send() 才能保持电机运转
        // ...
    }
}
```

### 多电机控制

```cpp
DShot motor1(&htim1, TIM_CHANNEL_1, DShot::DShotType::DSHOT600);
DShot motor2(&htim1, TIM_CHANNEL_2, DShot::DShotType::DSHOT600);
DShot motor3(&htim1, TIM_CHANNEL_3, DShot::DShotType::DSHOT600);
DShot motor4(&htim1, TIM_CHANNEL_4, DShot::DShotType::DSHOT600);

void armAll() {
    // 同时解锁所有电机
    uint32_t start = HAL_GetTick();
    while (HAL_GetTick() - start < 3100) { // 等待超过 3 秒
        motor1.send(0);
        motor2.send(0);
        motor3.send(0);
        motor4.send(0);
        for (volatile int k = 0; k < 100; k++);  // 短延时
    }
}

// 其余代码与之前相同
```

## API 参考

### 构造函数

```cpp
DShot(TIM_HandleTypeDef *htim, uint32_t channel, DShotType type)
```

- `htim` - 定时器句柄指针（例如 `&htim1`）
- `channel` - 定时器通道（例如 `TIM_CHANNEL_1`）
- `type` - DShot 速率：`DSHOT150`、`DSHOT300`、`DSHOT600`、`DSHOT1200`

### 方法

#### `bool begin()`
初始化 DShot 驱动。成功返回 `true`，失败返回 `false`。

#### `void disarm()`
执行解锁序列。**必须在电调上电后、发送油门值前调用**。阻塞约 3.1 秒。

#### `void send(uint16_t throttle)`
向电调发送油门值。
- `0` - 解锁命令（特殊命令）
- `1-47` - 特殊命令（蜂鸣、设置等）
- `48` - 0% 油门（最小值）
- `2047` - 100% 油门（最大值）

> **重要：** 必须持续调用（至少 100-1000 Hz）才能保持电机运转。如果约 100ms 未收到信号，电调将进入待机模式。

## 配置

### FreeRTOS 支持

要启用 FreeRTOS 非阻塞延时，请编辑 `dshot.h`：

```cpp
#define DSHOT_USE_FREERTOS true  // 设为 false 则仅使用 HAL_Delay
```

启用后，当 FreeRTOS 调度器运行时库会自动使用 `vTaskDelay()`，否则回退到 `HAL_Delay()`。

### 定时器时钟频率

如果你的系统时钟不同，请在 `dshot.h` 中调整定时器时钟频率：

```cpp
#define TIM_CLK_FREQ 168000000UL  // STM32F4 为 168 MHz
```

## 重要提示

1. **必须解锁** - 电调在正确使用 `disarm()` 解锁前不会响应油门命令

2. **需要持续更新** - 电调期望持续接收信号更新。如果停止发送值，电机将停止，电调进入待机模式（表现为发送油门信号，电机不转但电调低声蜂鸣）

3. **DMA 完成** - 库会自动处理 DMA 完成。传输过程中请勿手动禁用 DMA 或修改定时器 CCR 寄存器

## 致谢

- 原始 C 语言实现：[mokhwasomssi/stm32_hal_dshot](https://github.com/mokhwasomssi/stm32_hal_dshot)

## 许可证

本项目采用 MIT 许可证 - 详见 [LICENSE](LICENSE) 文件。
