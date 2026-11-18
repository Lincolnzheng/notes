# SPI 总线协议

SPI（Serial Peripheral Interface，串行外设接口）是一种同步串行通信接口。典型 SPI 由一个主机产生时钟并选择从机，数据可在 MOSI 和 MISO 上同时移动，因此常用于高速、板级、短距离通信。

SPI 只规定基本信号和移位时序，没有统一规定设备地址、寄存器命令、校验、最大事务长度或 CS 边界。传感器、Flash、屏幕等器件的上层命令必须查各自数据手册。

## 1. 信号组成

| 信号 | 全称 | 主机视角 | 作用 |
|---|---|---|---|
| SCK / SCLK | Serial Clock | 输出 | 主机提供串行时钟 |
| MOSI | Master Out, Slave In | 输出 | 主机发、从机收 |
| MISO | Master In, Slave Out | 输入 | 从机发、主机收 |
| NSS / SS / CS | Slave Select / Chip Select | 输出 | 选择从机，通常低有效 |

`NSS` 的 N 表示低电平有效，但并非所有器件都要求低有效或相同的片选时序，应以目标器件手册为准。

![[Pasted image 20260821150339.png]]

### 单主机、多从机连接

SCK、MOSI、MISO 可以共用，每个从机通常使用独立 CS：

```text
主机 SCK  ────────────── 从机1/从机2
主机 MOSI ────────────── 从机1/从机2
主机 MISO ◀───────────── 从机1/从机2
主机 CS1  ────────────── 从机1
主机 CS2  ────────────── 从机2
```

未选中的从机必须释放 MISO 为高阻态，否则多个从机可能同时驱动线路。若器件不能正确释放 MISO，应增加缓冲/开关或改用独立总线。菊花链只适用于明确支持该连接方式的器件，不是通用 SPI 行为。

## 2. SPI 为什么是全双工

主机和从机内部都可理解为移位寄存器。每出现一个时钟，双方同时移出一位、移入一位：

```text
主机移位寄存器 ──MOSI──▶ 从机移位寄存器
主机移位寄存器 ◀─MISO── 从机移位寄存器
                  SCK
```

因此：

- 主机发送命令时也会收到同样数量的数据，早期收到的内容可能无意义。
- 主机想读取 N 字节，仍必须产生 N 字节对应的时钟，通常通过发送 `0xFF` 或 `0x00` 等 Dummy 数据实现。
- “只发送”和“只接收”是软件关注点；在线路上是否双向同时移位取决于硬件模式。

STM32 还可能支持：

- 2-line Full Duplex：MOSI、MISO 分离，典型四线 SPI。
- 2-line Receive Only：双线结构下只关注接收。
- 1-line Half Duplex：单根双向数据线分时收发。
- Simplex：应用只连接一个数据方向。

不同系列和具体 SPI 实例支持的方向、数据宽度和 FIFO 能力不同。

## 3. 位顺序

![[Pasted image 20260821151227.png]]

- MSB（Most Significant Bit）：最高有效位。
- LSB（Least Significant Bit）：最低有效位。
- MSB First：先传最高有效位，是多数器件的常见默认值。
- LSB First：先传最低有效位。

主机和从机的位顺序必须一致。某些器件的数据由多个字节组成，还要额外区分“字节内位顺序”和“多字节数值的大小端”，两者不是同一个概念。

## 4. CPOL、CPHA 与四种 SPI 模式

- CPOL（Clock Polarity）决定 SCK 空闲电平。
- CPHA（Clock Phase）决定在第一个还是第二个有效边沿采样。

| SPI Mode | CPOL | CPHA | SCK 空闲 | 采样边沿 | 数据改变边沿 |
|---:|---:|---:|---|---|---|
| 0 | 0 | 0 | 低 | 上升沿（第 1 边沿） | 下降沿 |
| 1 | 0 | 1 | 低 | 下降沿（第 2 边沿） | 上升沿 |
| 2 | 1 | 0 | 高 | 下降沿（第 1 边沿） | 上升沿 |
| 3 | 1 | 1 | 高 | 上升沿（第 2 边沿） | 下降沿 |

STM32CubeMX 中通常对应：

- CPOL：`Low` / `High`。
- CPHA：`1 Edge` / `2 Edge`。

不要只凭“上升沿采样”填写 CPHA；第一个边沿是上升还是下降由 CPOL 决定。目标器件数据手册可能直接写 Mode 0~3，也可能只画时序图。

### 用逻辑分析仪判断模式

1. 观察 CS 无效时 SCK 的电平，判断 CPOL。
2. 找到 CS 有效后的第一个 SCK 边沿。
3. 检查第一位数据在第一个边沿前是否已经稳定。
4. 对照器件规定的采样边沿判断 CPHA。

若数据整体左移/右移一位，通常优先怀疑 CPHA、CS 建立时间或双方帧宽，而不是把某个字节简单加减一位补偿。

## 5. CS/NSS 不只是“选择设备”

许多从机把 CS 边沿当作一次事务的开始和结束：

```text
CS 拉低 → 命令 → 地址 → Dummy 周期 → 数据 → CS 拉高
```

- CS 拉低到第一个时钟之间要满足建立时间。
- 最后一个时钟到 CS 拉高之间要满足保持时间。
- 有些命令要求整个命令、地址和数据期间 CS 始终保持低电平。
- 有些器件靠 CS 拉高复位内部位计数器或提交写操作。

因此，不要假设分两次调用 HAL 函数时可以在中间随意拉高 CS。最通用的主机方案是把 NSS 配置为 Software，由普通 GPIO 控制每个器件的 CS；硬件 NSS 是否满足连续事务和多从机需求，应结合具体 STM32 SPI 版本确认。

## 6. 时钟频率

STM32 作为主机时，SPI SCK 一般由对应外设时钟经过预分频产生：

`fSCK = fPCLK / Prescaler`

F407 常见预分频为 2、4、8、16、32、64、128、256。F407 的 SPI1 位于 APB2，SPI2/3 位于 APB1；其他 F4 型号还可能提供更多 SPI 实例。计算前必须查看具体型号的时钟树。

最终频率必须同时满足：

- STM32 对该 SPI 实例的最高频率限制。
- 从机在当前供电、模式和读写命令下允许的最大频率。
- PCB 走线、负载、电平转换器和信号完整性要求。
- 从机数据输出延迟与主机输入采样余量。

通信不稳定时先降低 SCK。更高 GPIO Speed 只代表更快边沿，不等于总是更好；过快边沿可能增加振铃、串扰和 EMI。

## 7. STM32CubeMX 配置要点（以 F407 为例）

1. 选择 SPI 实例及 Master/Slave 模式。
2. 选择 2-line Full Duplex、1-line 等通信方向。
3. 配置 Data Size、First Bit、CPOL 和 CPHA，必须与从机一致。
4. 主机根据外设时钟与目标器件限制选择 Baud Rate Prescaler。
5. 决定 NSS 使用硬件还是软件；多从机和复杂事务通常使用 GPIO 手动控制 CS。
6. 设置 SCK/MOSI 为合适的复用推挽输出，MISO 为复用输入，并核对 AF 映射。
7. 使用中断时开启 SPI IRQ；使用 DMA 时配置 TX/RX DMA、DMA IRQ 和句柄链接。
8. 初始化完成后，用 HAL 传输接口发起通信。CubeMX 配置外设并不会自动产生业务数据。

> [!warning] 从机模式
> 从机不能产生 SCK，必须在主机拉低 CS 并开始时钟之前准备好待发送数据。主机时钟太快或软件/DMA 补充数据太慢，可能出现接收溢出或发送欠载。

## 8. 常用 HAL 接口

| 方式  | 发送                       | 接收                      | 同时收发                            |
| --- | ------------------------ | ----------------------- | ------------------------------- |
| 阻塞  | `HAL_SPI_Transmit()`     | `HAL_SPI_Receive()`     | `HAL_SPI_TransmitReceive()`     |
| 中断  | `HAL_SPI_Transmit_IT()`  | `HAL_SPI_Receive_IT()`  | `HAL_SPI_TransmitReceive_IT()`  |
| DMA | `HAL_SPI_Transmit_DMA()` | `HAL_SPI_Receive_DMA()` | `HAL_SPI_TransmitReceive_DMA()` |

SPI 天然全双工。需要严格控制 Dummy 字节和收到的数据位置时，`HAL_SPI_TransmitReceive()` 往往比把“发送”和“接收”想成完全独立操作更直观。

### 单寄存器读取示例

下面只演示常见结构；最高位为读标志是某些传感器的协议，不是 SPI 总线统一规定。

```c
#define SENSOR_CS_LOW()   HAL_GPIO_WritePin(SENSOR_CS_GPIO_Port, SENSOR_CS_Pin, GPIO_PIN_RESET)
#define SENSOR_CS_HIGH()  HAL_GPIO_WritePin(SENSOR_CS_GPIO_Port, SENSOR_CS_Pin, GPIO_PIN_SET)

/**
 * @brief 按“地址最高位为读标志”的器件协议读取一个寄存器。
 * @param reg 目标寄存器地址。
 * @param value 用于保存读回值的指针。
 * @return HAL 传输状态。
 */
HAL_StatusTypeDef Sensor_ReadRegister(uint8_t reg, uint8_t *value)
{
    uint8_t tx[2] = {(uint8_t)(reg | 0x80U), 0xFFU};
    uint8_t rx[2] = {0U, 0U};
    HAL_StatusTypeDef status;

    if (value == NULL)
    {
        return HAL_ERROR;
    }

    SENSOR_CS_LOW();
    status = HAL_SPI_TransmitReceive(&hspi1, tx, rx, 2U, 100U);
    SENSOR_CS_HIGH();

    if (status == HAL_OK)
    {
        *value = rx[1]; /* rx[0] 是发送命令期间同时移入的字节。 */
    }
    return status;
}
```

### 非阻塞调用规则

- `_IT` / `_DMA` 返回只表示启动结果，不表示线上传输结束。
- TX/RX 缓冲区（包括 [[DMA]] 缓冲区）在完成回调前必须有效，不能使用即将退出的局部数组。
- 软件 CS 必须在整个异步事务期间保持有效，在完成或错误回调中再释放。
- 完成回调包括 `HAL_SPI_TxCpltCallback()`、`HAL_SPI_RxCpltCallback()` 和 `HAL_SPI_TxRxCpltCallback()`。
- 错误通过 `HAL_SPI_ErrorCallback()` 处理，可使用 `HAL_SPI_GetError()` 获取错误位。

## 9. 常见问题与排查

| 现象 | 优先检查 |
|---|---|
| 完全没有时钟 | SPI 是否初始化、主机模式、传输 API 返回值、句柄是否正确 |
| 有时钟但数据全错 | CPOL/CPHA、位顺序、数据宽度、从机模式是否一致 |
| 数据整体移位一位 | CPHA、CS 建立时间、双方帧同步 |
| 读取恒为 `0xFF` | MISO 浮空/被上拉、CS 未选中、命令错误、从机未供电 |
| 读取恒为 `0x00` | MISO 短路/被下拉、器件复位、方向或引脚复用错误 |
| 第一个字节正确，后面错误 | CS 被提前切换、Dummy 数量、地址自增设置、处理不及时 |
| 单个从机正常，多从机异常 | CS 冲突、未选中设备未释放 MISO、模式/频率需切换 |
| 低速正常，高速异常 | 信号完整性、GPIO 边沿、走线、电平转换、从机时序上限 |
| HAL 返回 `HAL_BUSY` | 前一次异步传输未完成、状态机或错误恢复不完整 |
| 出现 OVR | 接收数据未及时取走，IRQ/DMA 配置或调用方式错误 |

SPI 本身没有 ACK。即使从机未连接，主机仍可能成功产生时钟并让 HAL 返回 `HAL_OK`；这只说明 STM32 的传输流程完成，不证明对端存在或数据有效。应用可读取芯片 ID、状态寄存器或使用协议 CRC 判断。

### 逻辑分析仪观察顺序

1. CS 是否选择了正确器件，低电平持续范围是否覆盖完整事务。
2. SCK 空闲电平与 CPOL 是否一致。
3. 数据在哪个边沿改变、在哪个边沿稳定。
4. 命令、地址、Dummy 周期和数据长度是否符合器件手册。
5. MOSI 与 MISO 是否接反，未选中的从机是否释放 MISO。
6. 实际 SCK 频率是否符合预期。

## 10. SPI 与 I²C 对比


| 项目 | SPI | [[IIC\|I²C]] |
|---|---|---|
| 数据线 | 通常 SCK、MOSI、MISO、每从机一个 CS | SCL、SDA 两线共享 |
| 双工 | 可全双工 | 通常半双工 |
| 寻址 | 总线不定义地址，常靠 CS | 协议内含地址 |
| 应答 | 无统一 ACK | 每字节有 ACK/NACK |
| 速度 | 通常更高 | 通常较低 |
| 引脚数量 | 多从机时 CS 增加 | 多设备仍可共用两线 |
| 电气方式 | 通常推挽 | 通常开漏加上拉 |

不能只按速度选择。还要看目标芯片支持、引脚数量、传输距离、软件复杂度、功耗和信号完整性。

## 11. 记忆要点

- SPI 没有统一地址和命令格式，先查从机数据手册。
- 主机产生 SCK；没有时钟就没有数据移动。
- 发一位的同时会收一位，读数据也需要发送 Dummy 数据来产生时钟。
- CPOL 决定空闲电平，CPHA 决定第一/第二边沿采样。
- CS 往往同时定义协议事务边界，不能随意切换。
- `HAL_OK` 不能证明从机存在，因为 SPI 没有 ACK。
- 异步传输完成前，缓冲区和 CS 状态都必须保持有效。

## 参考资料

- [ST AN5543：Guidelines for enhanced SPI communication](https://www.st.com/resource/en/application_note/an5543-enhanced-methods-to-handle-spi-communication-on-stm32-devices-stmicroelectronics.pdf)
- [ST：Getting started with SPI](https://wiki.st.com/stm32mcu/wiki/Getting_started_with_SPI)
- [ST STM32F4 HAL SPI 驱动头文件](https://github.com/STMicroelectronics/stm32f4xx-hal-driver/blob/master/Inc/stm32f4xx_hal_spi.h)
- [ST RM0090：STM32F405/407/427/429 参考手册](https://www.st.com/resource/zh/reference_manual/DM00031020.pdf)
