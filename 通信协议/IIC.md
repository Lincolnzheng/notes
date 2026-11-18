# I²C（IIC）总线协议

I²C（Inter-Integrated Circuit）是一种同步、串行、双向的两线总线。国内资料也常写作 IIC；STM32 外设、HAL 函数和官方手册统一使用 `I2C` 命名。

![[Pasted image 20260820160311.png]]

## 1. 核心特点

- 使用 `SCL` 时钟线和 `SDA` 数据线。
- 同一时刻由控制器产生时钟，数据在一根 SDA 上双向传输，因此通常称为半双工。
- 支持一条总线上连接多个目标设备，通过地址选择通信对象。
- 标准允许多个控制器，并定义仲裁机制；实际产品中常使用单控制器结构。
- 每个字节后都有 ACK/NACK，应答可以用来判断对方是否接受数据。
- 无独立片选线，布线较少，但速度通常低于 SPI。

常见速率：

| 模式 | 最大标称速率 | 说明 |
|---|---:|---|
| Standard-mode | 100 kbit/s | 常用低速模式 |
| Fast-mode | 400 kbit/s | STM32F4 常用高速配置 |
| Fast-mode Plus | 1 Mbit/s | 需要控制器和目标设备都支持 |
| High-speed mode | 3.4 Mbit/s | 有额外协议要求，并非所有 STM32 都支持 |

器件数量不仅受地址数量限制，还受引脚负载、总线电容、上拉电阻和时序限制。标准/快速模式经常见到 400 pF 的总线电容上限，但不同模式和器件可能不同，应以总线规范及器件数据手册为准。

> [!note] 术语
> 新版规范常使用 Controller/Target，旧资料和 STM32 HAL 使用 Master/Slave。本文讲协议时写“控制器/目标设备”，涉及 HAL 函数时保留 Master/Slave 原名。

## 2. 电气结构：为什么需要上拉电阻

SCL 和 SDA 通常采用开漏（Open-Drain）输出：设备只能主动拉低总线，释放后由外部上拉电阻把总线拉高。

```text
VDD ── Rpull-up ── SCL ── 各设备的开漏引脚
VDD ── Rpull-up ── SDA ── 各设备的开漏引脚
```

这样多个设备同时连接时不会出现一个强推高、另一个强拉低的直接冲突，并可实现 ACK、时钟延展和多控制器仲裁。总线空闲时 SCL、SDA 都为高电平。

![[Pasted image 20260820161122.png]]

### 上拉电阻如何选择

- 电阻过大：上升沿太慢，高速通信容易读错。
- 电阻过小：低电平电流增大，器件可能无法把电压拉到有效低电平，功耗也更高。
- 总线越长、设备越多、输入电容越大，通常需要更强的上拉。

近似上升时间关系为：

`tr ≈ 0.8473 × Rp × Cb`

其中 `Rp` 为上拉电阻，`Cb` 为总线总电容。设计时还要结合规范允许的最大上升时间、器件低电平灌电流能力和实际波形。常见的 4.7 kΩ 只是经验起点，不是所有电压、速率和总线都适用。

SDA 需要在输出和输入之间切换。开漏输出高电平实际上是“释放总线”，此时读取 GPIO 输入即可观察真实电平，不必像推挽输出那样频繁切换模式。

![[Pasted image 20260821113936.png]]

> [!warning]
> 上拉电源电压必须同时满足所有器件 I/O 电压范围。模块板上可能已经带上拉，多个模块并联会让等效阻值变小。

## 3. 基本时序

### 数据有效规则

- SCL 为高电平期间，SDA 应保持稳定，接收方在此期间读取数据。
- SCL 为低电平期间，发送方改变下一位数据。
- START 和 STOP 是例外，它们正是利用 SCL 为高时 SDA 的跳变来表示边界。

### START、Repeated START 与 STOP

| 名称 | 条件 |
|---|---|
| START（S） | SCL 为高时，SDA 从高变低 |
| Repeated START（Sr） | 不先产生 STOP，直接再次产生 START |
| STOP（P） | SCL 为高时，SDA 从低变高 |

Repeated START 可让控制器在不释放总线的情况下改变方向或重新寻址。读取寄存器时经常先写寄存器地址，再通过 Repeated START 切换为读。

### 字节与 ACK/NACK

I²C 一般先传最高有效位（MSB First）。每传 8 位数据，第 9 个时钟用于应答：

- ACK：接收方在第 9 个时钟把 SDA 拉低。
- NACK：接收方释放 SDA，总线保持高电平。

ACK 的产生者取决于方向：控制器写数据时由目标设备应答；控制器读数据时由控制器应答。连续读取时，控制器通常对中间字节回复 ACK，对最后一个字节回复 NACK，再产生 STOP，表示不再需要数据。

## 4. 常见传输序列

### 控制器写目标设备

```text
S → 7位地址 + W → ACK → 数据0 → ACK → 数据1 → ACK → P
```

### 控制器直接读取

```text
S → 7位地址 + R → ACK → 数据0 → ACK → 数据1 → NACK → P
```

### 读取目标设备内部寄存器

```text
S → 地址+W → ACK → 寄存器地址 → ACK
  → Sr → 地址+R → ACK → 数据... → NACK → P
```

这里的“寄存器地址”是目标芯片协议中的内部地址，不是 I²C 设备地址。有些芯片使用 8 位内部地址，有些使用 16 位，也可能要求命令字或特殊顺序，必须查看器件数据手册。

## 5. 7 位地址与读写位

最常见的地址格式是：

```text
线上第一个字节：A6 A5 A4 A3 A2 A1 A0 R/W
                   └──── 7位地址 ────┘ └1位┘
```

例如器件 7 位地址为 `0x68`：

- 写地址字节：`0xD0`，即 `(0x68 << 1) | 0`。
- 读地址字节：`0xD1`，即 `(0x68 << 1) | 1`。

数据手册可能写 `0x68`，也可能分别写 `0xD0/0xD1`，使用前先确认它表达的是 7 位地址还是已包含 R/W 位的线上字节。

### STM32F4 HAL 的地址写法

STM32F4 HAL 的 `DevAddress` 参数要求把数据手册中的 7 位地址左移一位，R/W 位由 HAL 自动处理：

```c
#define SENSOR_ADDR_7BIT   0x68U
#define SENSOR_ADDR_HAL    (SENSOR_ADDR_7BIT << 1U)
```

不要再手工把读写位 OR 进去，也不要把已经是 `0xD0` 的地址再次左移。其他平台或其他驱动接口可能直接接收 7 位地址，必须查看对应 API 文档。

## 6. 时钟延展与仲裁

### Clock Stretching

目标设备来不及处理时，可以持续拉低 SCL，让控制器等待。控制器和软件驱动必须支持并设置合理超时；并非所有器件都会使用该功能。

### 多控制器仲裁

多个控制器同时开始传输时，一边发送一边读取 SDA。发送高电平（释放）却读到低电平的一方丢失仲裁并停止驱动。仲裁依赖开漏“线与”特性。普通单控制器项目无需主动实现多主调度，但出现 `HAL_I2C_ERROR_ARLO` 时应考虑总线冲突或异常波形。

## 7. 硬件 I²C 与软件模拟 I²C

| 项目 | STM32 硬件 I²C | GPIO 模拟 I²C |
|---|---|---|
| 时序 | 外设自动生成 | 软件控制 GPIO |
| CPU 占用 | 较低，可使用中断/DMA | 通常较高 |
| 引脚 | 只能选择芯片支持的复用引脚 | 原则上可选普通 GPIO |
| 速度和一致性 | 通常更好 | 受延时函数和中断影响 |
| Clock Stretching/仲裁 | 由外设能力处理 | 必须在软件中正确读取线路并处理 |
| 调试 | 状态机复杂，需查看错误码 | 波形直观，但边界条件容易漏掉 |

软件模拟不等于天然稳定。开漏模拟、ACK 方向、时钟延展、超时、临界区以及引脚真实电平都需要正确实现。高速或后台传输优先考虑硬件 I²C。

## 8. STM32CubeMX 配置要点（以 F407 为例）


1. 选择 I2C 实例，并将 SCL/SDA 配为对应的 [[GPIO|复用开漏引脚]]。
2. 确认 GPIO 速度和外部上拉电阻；内部弱上拉通常不能替代合适的外部上拉。
3. 设置时钟速率、Fast-mode Duty Cycle、7/10 位寻址、自身地址等参数。
4. 控制器模式下通常不需要有效的 Own Address；目标设备模式才重点配置本机地址。
5. 使用中断时同时开启 Event IRQ 与 Error IRQ，并让生成的中断函数调用 HAL 处理函数。
6. 使用 DMA 时分别配置 TX/RX DMA、DMA IRQ 与句柄链接。
7. 调用 `HAL_I2C_Init()` 完成外设初始化，再启动具体传输。

> [!note]
> 不同 STM32 系列的 I²C 外设版本差别很大。F4 常用 `ClockSpeed/DutyCycle` 配置；较新的系列常使用 `Timing` 寄存器值，不能直接复制。

## 9. 常用 HAL 接口

| 需求     | 阻塞接口                        | 中断接口                | DMA 接口               |
| ------ | --------------------------- | ------------------- | -------------------- |
| 控制器发送  | `HAL_I2C_Master_Transmit()` | `..._Transmit_IT()` | `..._Transmit_DMA()` |
| 控制器接收  | `HAL_I2C_Master_Receive()`  | `..._Receive_IT()`  | `..._Receive_DMA()`  |
| 内部地址写入 | `HAL_I2C_Mem_Write()`       | `..._Write_IT()`    | `..._Write_DMA()`    |
| 内部地址读取 | `HAL_I2C_Mem_Read()`        | `..._Read_IT()`     | `..._Read_DMA()`     |
| 检测设备应答 | `HAL_I2C_IsDeviceReady()`   | —                   | —                    |

`Mem_Read/Mem_Write` 中的 Memory 是指“具有内部地址的 I²C 目标设备”，不限于 EEPROM。它们会处理内部地址阶段；是否适合某个传感器，仍取决于该器件协议。

对于要求“写内部地址 → Repeated START → 读取”的器件，优先使用合适的 `HAL_I2C_Mem_Read()` 或顺序传输 API。简单连续调用一次 `Master_Transmit()` 和一次 `Master_Receive()` 往往会在中间产生 STOP，不一定满足器件要求。

### 阻塞读取示例

```c
#define SENSOR_ADDR_HAL  (0x68U << 1U)

/**
 * @brief 从传感器的 8 位内部寄存器地址开始连续读取数据。
 * @param reg 起始寄存器地址。
 * @param data 接收缓冲区。
 * @param length 要读取的字节数。
 * @return HAL 传输状态。
 */
HAL_StatusTypeDef Sensor_Read(uint8_t reg, uint8_t *data, uint16_t length)
{
    return HAL_I2C_Mem_Read(&hi2c1,
                            SENSOR_ADDR_HAL,
                            reg,
                            I2C_MEMADD_SIZE_8BIT,
                            data,
                            length,
                            100U);
}
```

调用者仍需检查返回值。`HAL_OK` 表示本次总线操作成功，不代表读取值一定符合业务范围。

### 非阻塞调用规则


- `_IT` / `_DMA` 启动函数返回后，传输尚未完成。
- 缓冲区（包括 [[DMA]] 缓冲区）在完成回调前必须一直有效且不能被修改。
- 完成通知使用 `HAL_I2C_MasterTxCpltCallback()`、`HAL_I2C_MasterRxCpltCallback()`、`HAL_I2C_MemTxCpltCallback()` 或 `HAL_I2C_MemRxCpltCallback()`。
- 错误进入 `HAL_I2C_ErrorCallback()`；回调中优先记录状态或通知任务，不做长时间阻塞处理。
- 同一个句柄已有传输时再次启动，通常会返回 `HAL_BUSY`。

## 10. 常见错误与排查

| 现象 | 优先检查 |
|---|---|
| SCL、SDA 一直为高 | 外设是否启动、引脚复用是否正确、是否调用了传输 API |
| SDA/SCL 无法拉高 | 短路、目标设备掉电、上拉缺失/过小、器件锁住总线 |
| 地址阶段立即 NACK | 地址是否按 7 位/8 位正确换算、器件供电、地址脚、器件是否就绪 |
| 写入 ACK、读取数据错误 | 内部寄存器地址宽度、Repeated START、字节顺序、读取长度 |
| 100 kHz 正常，400 kHz 异常 | 上拉电阻、总线电容、走线、器件最大速率、时序参数 |
| HAL 返回 `HAL_BUSY` | 前一次传输未结束、BUSY 标志被占用、SDA 被拉低 |
| 偶发 `BERR/ARLO/AF` | 毛刺、供电、时序、仲裁、NACK 或中断延迟 |
| 中断/DMA 无回调 | EV/ER/DMA IRQ、句柄链接、中断函数是否转发给 HAL |

可使用 `HAL_I2C_GetError()` 查看错误位。F4 HAL 中常见错误包括：

- `HAL_I2C_ERROR_AF`：收到 NACK，可能是地址错误、设备忙或数据阶段被拒绝。
- `HAL_I2C_ERROR_BERR`：总线错误，例如异常 START/STOP。
- `HAL_I2C_ERROR_ARLO`：仲裁丢失。
- `HAL_I2C_ERROR_OVR`：接收/发送处理不及时。
- `HAL_I2C_ERROR_DMA`：DMA 传输异常。

若目标设备在传输中复位，可能一直拉低 SDA。先确认硬件没有短路；器件协议允许时，可把 SCL 临时切为开漏 GPIO，产生若干时钟帮助目标设备退出未完成字节，再生成 STOP 并重新初始化 I²C。恢复策略必须设置次数和超时，不能无限循环。

### 逻辑分析仪观察顺序

1. 空闲时两根线是否都为高。
2. START 是否正确。
3. 地址和 R/W 位是否符合预期。
4. 每个字节后的 ACK/NACK 是谁产生的。
5. 寄存器读取中是否出现 Repeated START。
6. 最后一个读字节后是否由控制器发 NACK 和 STOP。

## 11. 记忆要点

- 两根线都用开漏并通过电阻上拉。
- SCL 高时数据保持稳定；SCL 高时 SDA 的特殊跳变表示 START/STOP。
- 每 8 位后有第 9 个 ACK/NACK 时钟。
- 7 位地址与包含 R/W 位的线上地址字节不要混淆。
- STM32F4 HAL 通常接收 `7位地址 << 1`。
- 读取不是单纯“收数据”，许多器件先写内部地址，再 Repeated START 读回。
- 先看 HAL 返回值和错误码，再看逻辑分析仪波形。

## 参考资料

- [NXP UM10204：I²C-bus specification and user manual](https://www.nxp.com/docs/en/user-guide/UM10204.pdf)
- [ST：Getting started with I2C](https://wiki.st.com/stm32mcu/wiki/Getting_started_with_I2C)
- [ST STM32F4 HAL I2C 驱动](https://github.com/STMicroelectronics/stm32f4xx-hal-driver/blob/master/Src/stm32f4xx_hal_i2c.c)
