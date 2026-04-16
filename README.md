# DShot-STM32

English | [简体中文](README-zh-CN.md)

A C++ DShot protocol library for STM32 microcontrollers using HAL and DMA.

## Introduction

This library provides a C++ class-based implementation of the DShot protocol for controlling brushless ESCs (Electronic Speed Controllers) on STM32 microcontrollers. It uses STM32 HAL library with DMA for efficient, non-blocking motor control.

### Features

- **DMA-based transmission** - Non-blocking motor control
- **Object-oriented design** - Clean and reusable code
- **FreeRTOS support** - Optional non-blocking delays
- **Multiple DShot speeds** - DShot150, DShot300, DShot600, DShot1200

- **Tested on STM32F4** - Verified with BL_Heli32 DShot600 ESCs

## DShot Protocol Overview

DShot is a digital protocol for communicating with ESCs, offering advantages over traditional PWM:
- **Higher resolution** - 2000 steps (48-2047)
- **Error detection** - Built-in CRC checksum
- **No calibration needed** - Digital signal, precise throttle values
- **Bidirectional communication** - Telemetry support (optional)

### Data Frame Structure

![image](https://user-images.githubusercontent.com/48342925/105716366-0f1e2680-5f62-11eb-8d5c-651e15907a53.png)

| Bits | Purpose | Range |
|:----:|:--------|:------|
| 11 bits | Throttle / Command | 0-47: Commands, 48-2047: Throttle |
| 1 bit | Telemetry request | 0: No, 1: Yes |
| 4 bits | CRC Checksum | 0-15 |

### Bit Encoding

![image](https://user-images.githubusercontent.com/48342925/105717632-b780ba80-5f63-11eb-9508-54b4be15900d.png)

| | Bit 0 | Bit 1 |
|:---:|:---:|:---:|
| Duty Cycle | 37.425% | 74.850% |

### Transmission Speeds

| Mode | Bit Rate | Bit Time | Frame Time | Update Rate |
|:---:|:---:|:---:|:---:|:---:|
| DShot150 | 150 Kbit/s | 6.67 µs | 106.7 µs | ~9.4 kHz |
| DShot300 | 300 Kbit/s | 3.33 µs | 53.3 µs | ~18.8 kHz |
| DShot600 | 600 Kbit/s | 1.67 µs | 26.7 µs | ~37.5 kHz |
| DShot1200 | 1200 Kbit/s | 0.83 µs | 13.3 µs | ~75 kHz |

### Signal Example

<img src="https://user-images.githubusercontent.com/48342925/122077706-ec1fe080-ce36-11eb-9f24-a70c651b41c5.jpg" width="50%">

- Throttle value: 11
- Telemetry request: 0 (no)
- Checksum: 0111

### Arming Sequence

After power on, the ESC requires an arming sequence before it will respond to throttle commands:

1. **Power on ESC** - Wait for initialization 3 beeps
2. **Send zero throttle** - Continuously send `0` for ~3 seconds
3. **Wait for confirmation beep** - One high beep indicates successful arming
4. **Ready for throttle** - Can now send throttle values (48-2047)

<img src="https://user-images.githubusercontent.com/48342925/124644199-f466bb00-decc-11eb-8dc9-396d9b9706d7.png" width="70%">

## Hardware Requirements

- STM32 microcontroller (F1, F4, F7, H7 series)
- BLHeli_32 or compatible DShot ESC
- Brushless motor

## STM32CubeMX Configuration

### Timer Configuration

Configure a timer with PWM generation on the desired channels:

![MX_TIM_Config](/example-images/mxconfig%20(1).png)
![MX_TIM_Config](/example-images/mxconfig%20(2).png)

**Key settings:**
- Channel: PWM Generation Normal
- Prescaler: Will be set by library based on DShot speed
- Counter Period: 20-1 (for bit timing)

### DMA Configuration

Enable DMA for each timer channel used:

![MX_TIM_DMA_Config](/example-images/mxconfig%20(3).png)

**Key settings:**
- Direction: Memory to Peripheral
- Mode: Normal
- Data Width: Word (32-bit)

## Usage

1. Copy `src/dshot.h` and `src/dshot.cpp` to your project
2. Include `dshot.h` in your code
3. Configure STM32CubeMX as described above

### Basic Example

```cpp
#include "dshot.h"

// Create DShot instance for TIM1 Channel 1
DShot motor(&htim1, TIM_CHANNEL_1, DShot::DShotType::DSHOT600);

int main() {
    // Initialize motor
    if (!motor.begin()) {
        // Handle error
    }
    
    // Arm the ESC (required after power on)
    motor.disarm();
    
    // Now you can control the motor
    while (1) {
        // ...
        motor.send(100); // send() must be called continuously to keep the motor running
        // ...
    }
}
```

### Multiple Motors

```cpp
DShot motor1(&htim1, TIM_CHANNEL_1, DShot::DShotType::DSHOT600);
DShot motor2(&htim1, TIM_CHANNEL_2, DShot::DShotType::DSHOT600);
DShot motor3(&htim1, TIM_CHANNEL_3, DShot::DShotType::DSHOT600);
DShot motor4(&htim1, TIM_CHANNEL_4, DShot::DShotType::DSHOT600);

void armAll() {
    // Arm all motors simultaneously
    uint32_t start = HAL_GetTick();
    while (HAL_GetTick() - start < 3100) { // Wait for more than 3 seconds
        motor1.send(0);
        motor2.send(0);
        motor3.send(0);
        motor4.send(0);
        for (volatile int k = 0; k < 100; k++);  // Short delay
    }
}

// rest are the same as beforer
```

## API Reference

### Constructor

```cpp
DShot(TIM_HandleTypeDef *htim, uint32_t channel, DShotType type)
```

- `htim` - Pointer to timer handle (e.g., `&htim1`)
- `channel` - Timer channel (e.g., `TIM_CHANNEL_1`)
- `type` - DShot speed: `DSHOT150`, `DSHOT300`, `DSHOT600`, `DSHOT1200`

### Methods

#### `bool begin()`
Initialize the DShot driver. Returns `true` on success, `false` on failure.

#### `void disarm()`
Perform the arming sequence. **Must be called after ESC power on** and before sending throttle values. Blocks for ~3.1 seconds.

#### `void send(uint16_t throttle)`
Send a throttle value to the ESC.
- `0` - Disarm command (special command)
- `1-47` - Special commands (beep, settings, etc.)
- `48` - 0% throttle (minimum)
- `2047` - 100% throttle (maximum)

> **Important:** Must be called continuously (at least 100-1000 Hz) to keep the motor running. The ESC will enter standby mode if no signal is received for ~100ms.

## Configuration

### FreeRTOS Support

To enable FreeRTOS non-blocking delays, edit `dshot.h`:

```cpp
#define DSHOT_USE_FREERTOS true  // Set to false for HAL_Delay only
```

When enabled, the library will automatically use `vTaskDelay()` when the FreeRTOS scheduler is running, and fall back to `HAL_Delay()` otherwise.

### Timer Clock Frequency

Adjust the timer clock frequency in `dshot.h` if your system clock differs:

```cpp
#define TIM_CLK_FREQ 168000000UL  // 168 MHz for STM32F4
```

## Important Notes

1. **Arming is required** - The ESC will not respond to throttle commands until properly armed with `disarm()`

2. **Continuous updates required** - The ESC expects continuous signal updates. If you stop sending values, the motor will stop and the ESC will enter standby mode (indicated by sending throttle signal but motor not spinning and ESC beeping softly)

3. **DMA completion** - The library handles DMA completion automatically. Do not manually disable DMA or modify the timer CCR registers while transmitting

## Credits

- Original C implementation: [mokhwasomssi/stm32_hal_dshot](https://github.com/mokhwasomssi/stm32_hal_dshot)

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
