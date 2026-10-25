# TIM_Encoder：STM32 编码器控制 LED 调光

基于 **STM32F103C8T6** 和 STM32 HAL 库的旋转编码器调光示例。使用 TIM1 读取编码器 A/B 两路信号，通过 TIM3 输出 PWM 调节 LED 亮度，并通过按键切换当前控制的 LED。

## 功能介绍

- **旋转调光**：读取 TIM1 编码器计数，将控制值限制在 `0～100`，用于设置当前 PWM 通道的占空比。
- **切换 LED**：按下 PA11 按键，在 TIM3_CH2 和 TIM3_CH3 之间切换。切换时停止原通道的 PWM，再启动新通道。
- **串口观察**：通过 USART2 输出 `COUNT = 数值`，方便查看编码器计数。
- **循环更新亮度**：PWM 比较值在按键判断之外更新，旋转编码器即可调光，无需按键确认。

上电默认选择 **TIM3_CH2（PB5）**，计数和 PWM 比较值初始为 0。两个 LED 共用同一个亮度控制值，切换后新通道使用当前计数，程序没有分别保存两路亮度。

## 硬件与引脚配置

| 引脚 | 外设 / 功能 | 代码中的配置 | 接线说明 |
| --- | --- | --- | --- |
| PA8 | TIM1_CH1 | 输入，无内部上拉/下拉 | 编码器 A 相（模块常标为 CLK） |
| PA9 | TIM1_CH2 | 输入，无内部上拉/下拉 | 编码器 B 相（模块常标为 DT） |
| PA11 | 按键输入 | 内部上拉，低电平有效 | 接编码器按键 SW 或独立按键，按下时连接 GND |
| PB5 | TIM3_CH2 | 复用推挽输出 | 第一路 LED 的 PWM 信号 |
| PB0 | TIM3_CH3 | 复用推挽输出 | 第二路 LED 的 PWM 信号 |
| PA2 | USART2_TX | 复用推挽输出 | 接 USB 转串口模块的 RX |
| PA3 | USART2_RX | 输入，无内部上拉/下拉 | 接 USB 转串口模块的 TX；仅查看日志时可不接 |
| PA13 | SWDIO | SWD 调试接口 | 接 ST-Link 的 SWDIO |
| PA14 | SWCLK | SWD 调试接口 | 接 ST-Link 的 SWCLK |
| PD0 / PD1 | HSE 外部晶振 | 工程按 8 MHz 外部晶振配置 | 使用开发板上的晶振电路 |
| GND | 公共地 | — | 开发板、编码器和串口模块共地 |

### 接线说明

1. 编码器 A/B 两路当前没有启用 MCU 内部上拉。如果使用裸机械编码器或需要上拉的开漏输出模块，应提供外部上拉，或根据硬件修改输入上拉配置，保证输入电平稳定。
2. 外接 LED 时需串联合适的限流电阻。PWM 输出为高电平有效；对于引脚输出高电平时点亮的接法，占空比越大越亮。低电平点亮的 LED 电路，亮度变化方向相反。
3. TIM3 已启用**部分重映射**：CH2 输出在 **PB5**，CH3 输出在 **PB0**。接线时应以本工程配置为准。
4. 编码器和串口模块应使用与 MCU 兼容的 3.3 V 逻辑电平。编码器正转对应增加还是减少，取决于 A/B 接线及输入极性配置。

## 定时器配置

### TIM1：编码器计数

| 配置项 | 当前设置 |
| --- | --- |
| 输入通道 | CH1：PA8；CH2：PA9 |
| 编码器模式 | `TIM_ENCODERMODE_TI1` |
| 预分频寄存器 PSC | `2 - 1`，即 1 |
| 自动重装载值 ARR | 65535 |
| IC1 极性 | `TIM_ICPOLARITY_RISING` |
| IC2 极性 | `TIM_ICPOLARITY_FALLING` |
| IC1 / IC2 输入预分频 | `TIM_ICPSC_DIV1` |
| IC1 / IC2 数字滤波参数 | 15 |

程序通过以下语句启动编码器，并在主循环中轮询读取计数，不依赖编码器中断：

```c
HAL_TIM_Encoder_Start(&htim1, TIM_CHANNEL_ALL);
count = __HAL_TIM_GET_COUNTER(&htim1);
```

计数读取后的处理顺序为：

- 大于 6000：将控制值和 TIM1 计数器清零，用于处理从 0 向下计数回绕到 65535 附近的情况。
- 否则，大于 100：将控制值和 TIM1 计数器设置为 100。
- 其余情况：保留当前计数。

这是本示例的简单限幅逻辑。因此，在 0 处继续朝减小方向旋转，数值仍会保持为 0；在 100 处继续朝增大方向旋转，数值会保持为 100。

### TIM3：PWM 调光

| 配置项 | 当前设置 |
| --- | --- |
| 定时器输入时钟 | 72 MHz |
| 预分频寄存器 PSC | `72 - 1`，即 71 |
| 自动重装载值 ARR | `100 - 1`，即 99 |
| PWM 模式 | PWM mode 1，高电平有效 |
| PWM 通道 | CH2：PB5；CH3：PB0 |
| 初始比较值 | 0 |

当前时钟配置下，PWM 频率为：

```text
f_PWM = 72 MHz / (PSC + 1) / (ARR + 1)
      = 72 MHz / 72 / 100
      = 10 kHz
```

主循环将 `count` 写入当前通道的比较寄存器：

```c
__HAL_TIM_SET_COMPARE(&htim3, channel[channel_index], count);
```

`count` 为 `0～100`，对应输出高电平占空比约为 `0%～100%`。

## 按键与串口

PA11 使用内部上拉，按下为低电平。程序检测到按下后延时 10 ms 再次确认，切换 PWM 通道，然后等待按键松开，避免一次长按反复切换。等待松开期间主循环暂停，PWM 硬件继续输出，编码器硬件仍可计数。

串口终端设置：

| 参数 | 设置 |
| --- | --- |
| 串口外设 | USART2 |
| 波特率 | 115200 |
| 数据位 | 8 |
| 停止位 | 1 |
| 校验位 | 无 |
| 硬件流控 | 无 |

输出示例（实际数值取决于旋转情况）：

```text
COUNT = 0
COUNT = 1
COUNT = 2
```

主循环每轮包含 `HAL_Delay(100)`，因此正常情况下约每 100 ms 输出一次，实际间隔还包含串口发送和按键处理时间。当前应用使用串口打印计数，没有实现串口命令控制功能。

## 工程结构

```text
TIM_Encoder/
├── Core/
│   ├── Inc/                  # 应用头文件和 HAL 配置
│   └── Src/
│       ├── main.c            # 编码器读取、限幅、按键切换和 PWM 更新
│       ├── tim.c             # TIM1 编码器、TIM3 PWM 和引脚配置
│       ├── gpio.c            # PA11 按键输入配置
│       ├── usart.c           # USART2 初始化和 printf 字符输出
│       └── stm32f1xx_it.c     # 中断处理
├── Drivers/                  # CMSIS 和 STM32 HAL 驱动
├── cmake/                    # CMake 工具链及 CubeMX 构建配置
├── CMakeLists.txt
├── CMakePresets.json          # Debug / Release 构建预设
├── STM32F103xx_FLASH.ld       # 链接脚本
├── startup_stm32f103xb.s      # 启动文件
└── TIM_Encoder.ioc            # STM32CubeMX 配置文件
```

## 编译与运行

工程使用 C11、CMake 和 Ninja，工具链为 Arm GNU Toolchain（`arm-none-eabi-gcc`）。CubeMX 配置文件记录的版本为 STM32CubeMX 6.18.1、STM32Cube FW_F1 V1.8.7。

安装 CMake 3.22 或更新版本、Ninja 和 Arm GNU Toolchain，并将相关命令加入 `PATH`，在工程根目录执行：

```sh
cmake --preset Debug
cmake --build --preset Debug
```

需要 Release 构建时执行：

```sh
cmake --preset Release
cmake --build --preset Release
```

Debug 构建目标为 `build/Debug/TIM_Encoder.elf`。使用 ST-Link 配合支持 ELF 下载的 STM32 烧录或调试工具，将程序下载到目标板。

运行步骤：

1. 按引脚表连接编码器、两路 LED 和串口模块，并确认共地。
2. 下载程序，打开串口终端，设置为 `115200 8N1`。
3. 旋转编码器，观察 `COUNT` 和当前通道 LED 的亮度变化。
4. 按下并松开 PA11 按键，切换到另一路 LED，继续旋转调光。

## 排查提示

- **COUNT 变化但亮度不变**：检查 LED 接线和有效电平，确认 PWM 更新语句在按键判断之外，并确认使用 PB5 / PB0。
- **COUNT 始终为 0**：尝试向两个方向旋转，检查 PA8 / PA9 的接线、上拉和公共地；从 0 向减小方向旋转会被限幅为 0。
- **串口无输出**：确认 PA2 连接串口模块 RX、双方共地，并使用正确的串口号和波特率。

本文依据仓库中的代码和配置文件编写；实际 LED 亮度方向、编码器旋转方向和每格对应的计数变化取决于所接硬件。
