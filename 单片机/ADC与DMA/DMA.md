# DMA：直接存储器访问

DMA（Direct Memory Access，直接存储器访问）是负责搬运数据的硬件。CPU 配好源地址、目标地址、方向和数量后，DMA 可以在外设与内存之间搬运数据，不需要 CPU 逐个读取、写入。

> [!note] 本文范围
> 以 STM32F407 + STM32 HAL 为主要例子。F1 的 Channel、F4 的 Stream/Channel、部分新系列的 DMAMUX/GPDMA 并不相同，不能直接照搬资源编号。具体映射和可访问内存应查对应型号参考手册。

## 1. DMA 解决什么问题

以 ADC 采样为例：

- 不用 DMA：ADC 转换完成 → CPU 读数据寄存器 → CPU 存入数组。
- 使用 DMA：ADC 转换完成并发出 DMA 请求 → DMA 读数据寄存器 → DMA 存入数组 → 满足半满/全满条件时通知 CPU。

DMA 只负责搬运，不负责 ADC 转换、串口波特率、协议解析或计算平均值。它降低 CPU 搬运和逐字节中断的开销，但仍占用总线带宽，也需要 CPU 做初始化和后续处理。

| 方式  | CPU 如何参与           | 常见用途           |
| --- | ------------------ | -------------- |
| 轮询  | 主动等待标志，再读写数据       | 简单、低频、少量数据     |
| 中断  | 外设就绪时进入中断，由 CPU 搬运 | 少量或零散数据        |
| DMA | 硬件批量搬运，CPU 按块处理    | 连续采样、串口接收、大块发送 |

DMA 和中断不是二选一：**数据由 DMA 搬运，完成通知仍可以使用中断。**

## 2. 传输方向与配置参数

### 2.1 三种方向

| 方向 | 源 → 目标 | 例子 |
|---|---|---|
| Peripheral to Memory | 外设寄存器 → RAM | ADC 采样、UART RX |
| Memory to Peripheral | RAM → 外设寄存器 | UART TX、DAC 输出 |
| Memory to Memory | 存储区域 → 存储区域 | 内存复制；受控制器能力限制 |

这里的“外设”指芯片内部的 ADC、USART、SPI 等，不是指外接传感器。DMA 也不是随意读写所有地址的工具，例如不能靠普通内存复制代替 Flash 擦写流程。

### 2.2 CubeMX 中常见的参数

| 参数 | 含义 | ADC / UART RX 的常见选择 |
|---|---|---|
| Request / Channel / Stream | 外设请求与 DMA 资源的映射 | 由具体芯片映射表确定 |
| Direction | 搬运方向 | Peripheral to Memory |
| Peripheral Increment | 每次搬运后是否递增外设地址 | Disable：一直访问同一个数据寄存器 |
| Memory Increment | 每次搬运后是否递增内存地址 | Enable：依次写入数组 |
| Peripheral Data Width | 外设侧每次访问的宽度 | ADC 常用 Half Word；8 位 UART 常用 Byte |
| Memory Data Width | 内存侧每次访问的宽度 | 入门时通常与外设侧一致 |
| Mode | 完成指定数量后是否自动重装 | Normal / Circular |
| Priority | DMA 请求之间的仲裁优先级 | 按带宽和实时性要求选择 |

Byte = 8 bit，Half Word = 16 bit，Word = 32 bit。地址递增的步长与配置的数据宽度有关。

**传输数量不一定是字节数。** 例如等宽 Half Word 配置下，搬运 100 个数据项占用 200 byte；不能把 `sizeof(uint16_t数组)` 不加换算地当作 ADC 的转换结果个数。

DMA Priority 与 NVIC 中断优先级是两套东西：前者影响搬运仲裁，后者影响 CPU 响应中断的先后。

### 2.3 F407 的 Stream 与 Channel

F407 的 DMA1、DMA2 各有 8 个 Stream；一个 Stream 保存一套传输配置，Channel 用于选择该 Stream 对应的请求来源。**DMA Channel 与 ADC 的模拟输入通道不是同一个概念。**

同一 Stream 不能同时服务两个独立传输。F407 的内存到内存传输使用 DMA2，且不能使用循环模式。FIFO、突发和双缓冲属于进阶配置，应按宽度、对齐及控制器限制使用。参考 [ST AN4031：DMA 控制器](https://www.st.com/resource/en/application_note/an4031-using-the-stm32f2-stm32f4-and-stm32f7-series-dma-controller-stmicroelectronics.pdf)。

## 3. Normal 与 Circular

### Normal：普通模式

搬运指定数量后停止本次 DMA 传输。下一次需要软件重新启动。

- 适合一次性采样、发送一块固定数据。
- DMA 停止不代表外设一定停止，例如 ADC 可能仍继续转换。
- 软件重新启动接收期间存在空窗，持续高速数据可能丢失。

### Circular：循环模式

搬运到缓冲区末尾后，硬件自动重新装载计数和地址，继续从头搬运。

- 适合连续 ADC 采样、连续 UART 接收。
- 不需要每次全满都调用一次启动函数。
- **DMA 不会等待 CPU 读完，旧数据可能被新数据覆盖。**

循环 DMA 也不等于软件环形缓冲区：前者循环写同一块内存，后者通常还维护读写位置、空满状态及溢出策略。若把 DMA 数据复制到软件环形缓冲区，应只复制新增部分，并定义满缓冲时如何处理。

## 4. ADC 的三个模式不要混淆

原笔记中的“循环模式、持续转换模式、DMA 连续请求”控制的是三个不同环节。

| 配置 | 属于谁 | 决定什么 |
|---|---|---|
| Continuous Conversion Mode | ADC | 一轮规则序列结束后，是否自动开始下一轮转换 |
| DMA Continuous Requests | ADC | 是否允许 DMA 请求持续产生 |
| DMA Circular Mode | DMA | 传输计数用完后，是否自动回到缓冲区开头继续搬运 |

以 F407 的规则组为例：

- 软件启动后持续自由采样：通常开启 ADC 连续转换、ADC DMA 连续请求，并使用 DMA Circular。
- 定时器固定频率采样：通常关闭 ADC 连续转换，配置定时器 TRGO 触发；长期采集时仍可开启 DMA 连续请求并使用 Circular。先准备 ADC + DMA，再启动触发定时器。
- 只采一批：使用 Normal，并设计好 ADC/触发源的停止条件，不要让 DMA 停止后 ADC 仍无节制地产生结果。

扫描模式（Scan）则决定是否按 Rank 顺序转换多个通道；它不等于连续模式。若 Rank 1/2/3 分别配置三个通道，DMA 数组中就按这三个 Rank 的顺序重复存放结果。

> [!warning]
> 不要为了“持续采样”把所有选项都打开。定时器触发场景若开启连续转换，第一次触发后可能转为自由连续转换，采样节奏就不再由每次定时器事件决定。

这些字段及连续请求与循环 DMA 的配合限制可查 [ST F4 HAL ADC 配置定义](https://github.com/STMicroelectronics/stm32f4xx-hal-driver/blob/master/Inc/stm32f4xx_hal_adc.h)。其他系列可能采用不同字段。

## 5. CubeMX / HAL 配置顺序

1. 配置外设本身：ADC 通道、Rank、采样时间和触发源，或 UART 波特率、数据位等。
2. 添加对应 DMA 请求，检查 Stream/Channel 是否与其他外设冲突。
3. 设置方向、地址递增、数据宽度、Normal/Circular。
4. 开启 DMA 时钟并配置 DMA NVIC；若功能依赖外设中断，也要开启外设 IRQ。
5. 检查 HAL 句柄关联，例如 ADC 的 `__HAL_LINKDMA(hadc, DMA_Handle, hdma_adc1)`、UART RX 的 `__HAL_LINKDMA(huart, hdmarx, hdma_usart2_rx)`。
6. 完成初始化后，手动调用 `HAL_ADC_Start_DMA()` 或 UART 的 DMA 启动接口；CubeMX 配好 DMA 不等于已经开始搬运。
7. 在完成/错误回调中记录状态，在主循环或任务中处理数据。

> [!tip] 初始化顺序
> DMA 时钟必须在 `HAL_DMA_Init()` 访问寄存器之前开启。CubeMX 工程通常先调用 `MX_DMA_Init()`，再初始化会在 MSP 中配置 DMA 的 ADC/UART；最终应检查生成代码的实际调用顺序，不只看函数名字。

使用 ADC/UART 的 DMA 功能时优先调用外设 HAL 接口，它会设置内部回调和外设 DMA 请求。不要同时再对同一个 DMA 句柄手动调用 `HAL_DMA_Start_IT()`，也不要随意覆盖 HAL 安装的 DMA 内部回调。

## 6. ADC DMA 的使用与完成通知

### 6.1 启动接口的三个参数

下面是 F407 单 ADC、12 位右对齐、DMA 两侧均为 Half Word、Normal 模式的一次性采集片段。需已有 CubeMX 生成的 `hadc1` 和 DMA 配置，不是完整工程。

```c
/* 缓冲区需在 DMA 完成前一直有效，并放在 DMA 可访问的 SRAM。 */
static uint16_t adc_samples[96];
static volatile uint8_t adc_ready;
static volatile uint8_t adc_failed;

/** 启动一批采集；仅在上一批采集及数据处理结束后调用。 */
HAL_StatusTypeDef ADC_StartBatch(void)
{
    adc_ready = 0U;
    adc_failed = 0U;
    return HAL_ADC_Start_DMA(&hadc1,
                             (uint32_t *)adc_samples,
                             sizeof(adc_samples) / sizeof(adc_samples[0]));
}

/** ADC DMA 全传输完成回调：只通知，不在中断里处理整批数据。 */
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc == &hadc1)
    {
        adc_ready = 1U;
    }
}

/** ADC 错误回调：通知主循环检查错误并按应用流程恢复。 */
void HAL_ADC_ErrorCallback(ADC_HandleTypeDef *hadc)
{
    if (hadc == &hadc1)
    {
        adc_failed = 1U;
    }
}
```

- 第一个参数：ADC 句柄。
- 第二个参数：目标缓冲区地址。F4 HAL 接口声明为 `uint32_t *`，这里的强制转换不意味着 DMA 一定按 32 位搬运；实际宽度由 DMA 配置决定。
- 第三个参数：本次搬运的转换结果数量，不是毫秒，也不是固定含义的字节数。

主循环需要检查启动返回值；看到 `adc_ready` 后读取数组，处理完成后再安排下一批。若 ADC 使用外部触发，还必须保证触发源确实运行；停止采集时应协调触发源与 `HAL_ADC_Stop_DMA()`。

原来的“无法判断是否转换完成”应改成：**启动函数返回只表示启动操作的结果，不表示整个缓冲区已经采集完成；使用 DMA 完成回调获取通知。** 使用 ADC DMA 时，通常不再逐点用 `HAL_ADC_GetValue()` 读取结果，而是读 DMA 数组。接口行为参考 [ST F4 HAL ADC 源码](https://github.com/STMicroelectronics/stm32f4xx-hal-driver/blob/master/Src/stm32f4xx_hal_adc.c)。

### 6.2 回调是怎么来的

```text
DMA 达到传输条件
  → DMAx_Streamy_IRQHandler()
  → HAL_DMA_IRQHandler(&对应DMA句柄)
  → HAL 内部的 ADC DMA 回调
  → HAL_ADC_ConvHalfCpltCallback() / HAL_ADC_ConvCpltCallback()
  → 设置标志或通知任务
  → 主循环 / 任务处理数据
```

中断入口名称取决于实际 DMA 资源，不能固定照抄某个 Stream。多个 ADC 共用同名 HAL 回调时，要检查句柄区分实例；已有回调就在原函数中合并处理，避免重复定义。

### 6.3 半满和全满如何配合

以 96 个数据项的循环缓冲区为例：

- HT（Half Transfer）：前 48 项完成，可处理前半区，此时 DMA 开始写后半区。
- TC（Transfer Complete）：后 48 项完成，可处理后半区，此时 DMA 回到前半区。
- CPU 必须在 DMA 再次写入该半区之前处理完，或及时复制到独立缓冲区。

若三通道扫描，两半都应包含完整扫描序列，因此总长度可选 `2 × 通道数 × 每半区采样轮数`。

对于速率稳定的采样，若总缓冲区有 N 个数据项、每秒写入 R 个数据项，则每半区的处理窗口约为 `N / (2 × R)` 秒。R 是所有通道合计的数据项速率，不一定等于每通道采样率。

单个布尔标志会合并连续事件；循环采样应考虑事件计数、队列、半区所有权及处理超时检测。`volatile` 不能保证不会覆盖，也不能代替这些机制。

## 7. UART DMA 与 IDLE


| HAL 接口                           | 用途                          |
| -------------------------------- | --------------------------- |
| `HAL_UART_Transmit_DMA()`        | DMA 发送指定数量的数据               |
| `HAL_UART_Receive_DMA()`         | DMA 接收指定数量的数据               |
| `HAL_UARTEx_ReceiveToIdle_DMA()` | 接收数据，并支持串口空闲事件通知；需 HAL 版本支持 |

### 7.1 IDLE 不等于收到一帧协议

例如，[[modbus协议|Modbus RTU]] 使用帧间静默时间分隔报文，不能只靠 UART 的 IDLE 事件判断整帧结束。

IDLE 检测的是 [[USART|UART]] 接收线在字符接收后的空闲，时间量级约为一个字符。它不认识 Modbus 的长度、CRC 或 RTU t3.5。一次 IDLE 可能对应半帧，也不能保证每次 DMA 回调都是一帧；协议层仍需自己判断边界。

`HAL_UARTEx_RxEventCallback(huart, Size)` 还可能由 HT、TC 触发，支持时可用 `HAL_UARTEx_GetRxEventType()` 区分。

### 7.2 Normal 与 Circular 的处理不同

| 场景 | 处理原则 |
|---|---|
| Normal + IDLE/TC | 本轮接收结束后重新启动；重新挂载存在接收空窗 |
| Normal + HT | 接收仍在进行，不要误当结束并重新启动 |
| Circular + HT/TC/IDLE | 接收继续运行，通常只处理新数据，不反复启动 DMA |

`Size` 表示缓冲区内截至此次事件的有效位置，不应普遍理解成“距上次新增字节数”。Circular 模式需保存上次位置、计算新增区间并处理回绕；同一位置的重复通知不能重复消费。若 DMA 已完整绕过一圈，仅靠位置差无法发现丢失，必须保证处理及时或增加溢出检测。参见 [ST F4 HAL UART 源码及 RxEvent 说明](https://github.com/STMicroelectronics/stm32f4xx-hal-driver/blob/master/Src/stm32f4xx_hal_uart.c)。

### 7.3 工程中还要注意

- 接收回调里只做有界的数据搬运和通知，CRC、协议解析、打印放到主循环。
- DMA IRQ 与 UART IRQ 都要正确转发给 HAL；IDLE 依赖 UART IRQ，不是只打开 DMA 中断就够了。
- 异步发送期间，TX 缓冲区必须有效且不能被修改。函数返回不等于发送完成。
- DMA 搬完最后一项不等于最后停止位已离开发送引脚。RS-485 DE 应按 UART 真正发送完成的时机释放；不要直接在 DMA TC 时拉低。

## 8. 缓冲区、内存与 Cache


1. **生命周期**：不要把即将退出函数的局部数组交给异步 DMA；使用长期有效的缓冲区。
2. **[[内存管理|内存可达性]]**：F407 的 CCM RAM 不能作为 DMA1/DMA2 的数据缓冲区。CPU 能访问某块 RAM，不代表 DMA 也能；跨系列必须查总线连接关系。
3. **宽度与容量**：数组类型、数据对齐、DMA 宽度、传输数量必须匹配，避免越界或错位。
4. **并发访问**：不要在 DMA 正写某一块数据时把它当作稳定快照，也不要修改 DMA 正在发送的数据。
5. **Cache 一致性**：F7/H7 等带 D-Cache 的芯片，要额外处理 CPU Cache 与 DMA 所见内存的一致性。

对使用可缓存内存的 DMA，TX 启动前通常需要 Clean，使 CPU 修改写回 RAM；RX 需要在 DMA 接管缓冲区前安排合适的 Cache 维护，并在完成后、CPU 读取前 Invalidate。维护范围须按 Cache line 对齐，不能误伤同一缓存行中的其他变量；DMA 活动期间也不能随意访问该缓冲区。可按设计使用 MPU 非缓存区简化管理。具体策略参考 [ST AN4839：F7/H7 Cache 一致性](https://www.st.com/resource/en/application_note/DM00272913-.pdf)。

> [!warning]
> `volatile` 只影响编译器对访问的优化，不会自动清理 Cache，也不提供缓冲区所有权、原子操作或防覆盖能力。

## 9. 常见问题排查

| 现象                | 优先检查                                            |
| ----------------- | ----------------------------------------------- |
| DMA 启动失败或一直 BUSY  | 上次是否结束、返回值、句柄状态、是否重复启动                          |
| 启动成功但数组不变         | 外设触发源、DMA 请求使能、Stream/Channel 映射、内存可达性          |
| 只有数组第一个元素变化       | Memory Increment 是否开启                           |
| 数据错位或数组越界         | 数据宽度、对齐、Length 是否误传字节数                          |
| 只采集一次就停止          | Normal/Circular、ADC 连续请求、触发源是否继续工作              |
| 有数据但没有完成回调        | 数量是否达到、DMA NVIC、IRQ 是否调用 `HAL_DMA_IRQHandler()` |
| ADC Overrun       | ADC 产生数据太快、DMA 未持续搬运、总线争用或停止顺序                  |
| UART 短包无通知        | 是否使用 IDLE、UART IRQ 是否开启、HAL 是否支持对应接口            |
| 串口数据重复或丢失         | HT/TC/IDLE 区分、Size 增量计算、重启空窗、循环缓冲覆盖             |
| F4 正常，换 F7/H7 后异常 | RAM 区域是否可达、Cache 维护、缓冲区对齐                       |



## 10. 记忆要点

- 外设负责产生/消耗数据，DMA 负责搬运，CPU 负责配置和处理。
- 外设地址一般固定，内存地址一般递增。
- Length 先确认单位，不能一律理解为字节数。
- ADC 连续转换、ADC DMA 连续请求、DMA 循环模式分别控制不同环节。
- DMA 全满不代表 CPU 已经处理完，UART DMA TC 也不代表串口线已发完。
- Circular 不等待消费者，IDLE 不代替协议分帧。
