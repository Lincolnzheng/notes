# Flash：原理与 STM32 HAL 函数

Flash 是掉电后仍能保存数据的非易失性存储器。STM32 的程序通常存放在片内 Flash，也可以划出独立区域保存设备地址、校准系数、配置参数等。

> [!note] 适用范围
> 本文以 **STM32F407、VDD 为 2.7～3.6 V、32 位编程**为例。F1 的页擦除、F4 的扇区擦除及 H7 的 Flash Word 编程不同，擦写配置不能直接跨系列使用。SPI Flash（如 W25Qxx）需要 SPI 驱动和器件命令，不使用本文的片内 Flash HAL。

## 1. Flash 与 RAM 的区别

| 项目 | SRAM | 片内 Flash |
|---|---|---|
| 掉电数据 | 丢失 | 保留 |
| 主要用途 | 栈、堆、变量、缓冲区 | 程序、常量、持久化参数 |
| 读取 | 地址访问 | 内存映射地址访问 |
| 写入 | 普通赋值 | 解锁后通过编程操作写入 |
| 修改旧数据 | 可直接覆盖 | 通常需要先擦除所在扇区 |
| 擦写寿命 | 无 Flash 式擦除周期限制 | 有限，按具体数据手册设计 |

Flash 擦除后每位为 1，因此字节读作 `0xFF`，32 位字读作 `0xFFFFFFFF`。编程能把位从 1 改为 0；恢复到 1 需要擦除。实际还受编程粒度、对齐和重复编程规则限制，不能仅凭位运算判断某次重复写入是否允许。

使用时应先确认目标区域的擦除状态。`HAL_FLASH_Program()` 不会自动擦除扇区。

## 2. F407 Flash 的地址与扇区

程序 Flash 从 `0x08000000` 开始。以下为 **1 MiB 容量型号**的扇区布局；512 KiB 型号只有 Sector 0～7。请通过完整料号、数据手册及容量寄存器确认实际容量。

| 扇区  | 起始地址         | 结束地址（包含）     | 大小      |
| --- | ------------ | ------------ | ------- |
| 0   | `0x08000000` | `0x08003FFF` | 16 KiB  |
| 1   | `0x08004000` | `0x08007FFF` | 16 KiB  |
| 2   | `0x08008000` | `0x0800BFFF` | 16 KiB  |
| 3   | `0x0800C000` | `0x0800FFFF` | 16 KiB  |
| 4   | `0x08010000` | `0x0801FFFF` | 64 KiB  |
| 5   | `0x08020000` | `0x0803FFFF` | 128 KiB |
| 6   | `0x08040000` | `0x0805FFFF` | 128 KiB |
| 7   | `0x08060000` | `0x0807FFFF` | 128 KiB |
| 8   | `0x08080000` | `0x0809FFFF` | 128 KiB |
| 9   | `0x080A0000` | `0x080BFFFF` | 128 KiB |
| 10  | `0x080C0000` | `0x080DFFFF` | 128 KiB |
| 11  | `0x080E0000` | `0x080FFFFF` | 128 KiB |

扇区大小不相等，不能用统一的“页大小”计算 F407 的所有擦除地址。擦除 Sector 7 会删除其中全部 128 KiB，而不只是准备修改的几个字节。布局依据 [ST RM0090 Flash 章节](https://www.st.com/resource/en/reference_manual/dm00031020-stm32f405xx-stm32f407xx-advanced-arm-based-32-bit-microcontrollers-stmicroelectronics.pdf)。

系统存储器（ST 出厂 Bootloader）、OTP、Option Bytes 与用户程序 Flash 用途不同，本文主要介绍用户程序 Flash 的读、擦除和编程操作。

## 3. 常用 HAL 函数

| 接口                                         | 作用                | 关键点                          |
| ------------------------------------------ | ----------------- | ---------------------------- |
| `HAL_FLASH_Unlock()`                       | 解锁 Flash 控制寄存器    | 检查返回值                        |
| `HAL_FLASH_Lock()`                         | 锁定 Flash 控制寄存器    | 擦写结束及失败路径均应执行                |
| `HAL_FLASHEx_Erase(&erase, &sector_error)` | 阻塞擦除              | 使用扇区编号和数量；失败扇区通过输出参数返回       |
| `HAL_FLASH_Program(type, address, data)`   | 阻塞编程              | 写一次所选宽度的数据；`data` 是数值，不是数据指针 |
| `HAL_FLASH_GetError()`                     | 查询最近 HAL Flash 错误 | 应在失败后立即记录                    |
| `HAL_FLASH_Program_IT()`                   | 中断方式编程            | 返回不代表操作结束                    |
| `HAL_FLASHEx_Erase_IT()`                   | 中断方式擦除            | 配合 Flash IRQ 和完成/错误回调        |

常见编程类型为 BYTE、HALFWORD、WORD、DOUBLEWORD。本文使用 `FLASH_TYPEPROGRAM_WORD`，每次写 4 byte，目标地址必须 4 字节对齐。**API 的 Data 参数虽然是 `uint64_t`，不代表一定写 8 字节。**

F407 的 x32 编程对应 2.7～3.6 V 条件，擦除时使用 `FLASH_VOLTAGE_RANGE_3`。x64 还涉及外部 VPP 等条件，不能只因为有 DOUBLEWORD 宏就使用。具体规则见 [RM0090 的 Programming parallelism](https://www.st.com/resource/en/reference_manual/dm00031020-stm32f4xxx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)。

Flash 通常不需要单独创建句柄或调用 `MX_FLASH_Init()`。读取也无需解锁，可直接读取映射地址：

```c
uint32_t value = *(volatile const uint32_t *)0x08060000U;
```

接口和错误处理依据 [ST F4 HAL Flash 实现](https://github.com/STMicroelectronics/stm32f4xx-hal-driver/blob/master/Src/stm32f4xx_hal_flash.c)及 [Flash 扩展驱动](https://github.com/STMicroelectronics/stm32f4xx-hal-driver/blob/master/Src/stm32f4xx_hal_flash_ex.c)。

## 4. HAL 函数参数与使用顺序

### 解锁和锁定

```c
HAL_StatusTypeDef HAL_FLASH_Unlock(void);
HAL_StatusTypeDef HAL_FLASH_Lock(void);
```

解锁后才能配置擦除、编程操作。操作完成后重新锁定；失败路径也要处理锁定。锁定控制寄存器不影响正常读取，也不等于设置了 Flash 读保护。

### 擦除：HAL_FLASHEx_Erase

```c
HAL_StatusTypeDef HAL_FLASHEx_Erase(
    FLASH_EraseInitTypeDef *pEraseInit,
    uint32_t *SectorError);
```

| 参数/字段 | 含义 |
|---|---|
| `pEraseInit` | 指向擦除配置结构体 |
| `TypeErase` | 扇区擦除选 `FLASH_TYPEERASE_SECTORS`；全片/Bank 擦除属于另一种操作 |
| `Sector` | 起始扇区编号，例如 `FLASH_SECTOR_7`，不是地址 |
| `NbSectors` | 从起始扇区开始连续擦除多少个扇区 |
| `VoltageRange` | 按实际供电选择擦除并行度，例如 `FLASH_VOLTAGE_RANGE_3` |
| `SectorError` | 输出失败扇区信息；没有报告失败扇区时为 `0xFFFFFFFFU`，仍需检查函数返回值 |

例如，选择 Sector 7 且数量为 1，会擦除整个 128 KiB 扇区。HAL 不会自动保留里面的其他参数。

### 编程：HAL_FLASH_Program

```c
HAL_StatusTypeDef HAL_FLASH_Program(
    uint32_t TypeProgram,
    uint32_t Address,
    uint64_t Data);
```

| 参数 | 含义 |
|---|---|
| `TypeProgram` | 编程宽度，例如 `FLASH_TYPEPROGRAM_WORD` |
| `Address` | 要编程的 Flash 绝对地址，必须满足宽度和地址范围要求 |
| `Data` | 本次写入的数值；Word 模式使用低 32 位 |

这个函数执行一次编程操作。写数组时需要按编程宽度逐次调用，并检查每次的结果。不能把数组首地址作为 `Data` 来代表“写整个数组”。

### 返回值与错误信息

阻塞操作需要检查 `HAL_OK`、`HAL_ERROR`、`HAL_BUSY` 或 `HAL_TIMEOUT` 等返回状态；失败时及时读取 `HAL_FLASH_GetError()`，记录详细错误位。错误位可能为 0，例如某些忙或超时情况，因此不能只检查错误码而忽略返回值。

### 中断方式

`HAL_FLASH_Program_IT()` 和 `HAL_FLASHEx_Erase_IT()` 启动操作后，由 Flash 中断处理后续状态：

- Flash IRQ 入口转发到 `HAL_FLASH_IRQHandler()`。
- 操作通知使用 `HAL_FLASH_EndOfOperationCallback()`。
- 错误通知使用 `HAL_FLASH_OperationErrorCallback()`。
- 多扇区擦除可能有阶段通知，不能把每次回调都理解为整个请求完成，应按对应 HAL 版本解释回调参数。

不要在异步操作刚启动后立即锁定或发起下一次擦写。应在确认操作结束后收尾，并对多个任务的 Flash 访问进行串行化。

### 基本顺序

1. 确认芯片型号、供电、目标地址及所在扇区。
2. 确认该区域已经通过链接配置预留，备份需要保留的数据。
3. 调用 `HAL_FLASH_Unlock()` 并检查结果。
4. 需要擦除时，配置结构体并调用 `HAL_FLASHEx_Erase()`。
5. 按所选宽度调用 `HAL_FLASH_Program()`。
6. 处理返回值、错误信息和必要的缓存维护，读回校验。
7. 调用 `HAL_FLASH_Lock()`。

读取无需执行上述解锁和擦写流程，直接访问有效的映射地址即可。

## 5. 片内 Flash 与片外 Flash


| 类型             | 访问路径                            | 常用接口                           |
| -------------- | ------------------------------- | ------------------------------ |
| STM32 片内 Flash | CPU → 片内 Flash 控制器              | 擦写使用 `HAL_FLASH_*`；读取使用内存映射地址  |
| 片外串行 Flash     | CPU → SPI/QSPI/OSPI 控制器 → 外部芯片  | 对应通信 HAL，加上器件要求的命令             |
| 片外并行 Flash     | CPU → 支持该器件的 FSMC/FMC 接口 → 外部芯片 | 总线接口及器件驱动，操作规则取决于 NOR/NAND 等类型 |

片外 Flash 也可以有现成的器件驱动或开发板 BSP。通信 HAL 负责传输，器件驱动负责组织写使能、命令、地址、数据和忙状态查询等步骤；`HAL_FLASH_Program()` 不会替外部 [[SPI]] Flash 完成这些操作。

部分外部存储控制器支持内存映射读取。配置完成后，CPU 可以通过映射地址读取外部 Flash，但底层仍由控制器发出总线事务；擦除和编程仍需遵守器件命令与控制器规则。

## 6. 参数区预留


参数区应与程序（含 [[IAP]] 的应用区）、常量以及初始化数据的 Flash 装载区域分开，并按完整擦除单元规划。检查链接脚本或 Keil scatter 配置，以及最终 `.map` 文件。

**在 C 文件中定义一个存储地址，不会让链接器自动避开该区域。** 参数区与程序重叠时，擦除会破坏固件。下载器的全片擦除也会删除参数，需要同时确认烧录策略。

## 7. 实际项目的参数保存

- **不要每次修改一个参数都擦整扇区**：可先在 RAM 修改，用户确认或延时合并后再保存。
- **更新扇区中部分内容**：需要先保留同扇区其他有效数据；单扇区“读出—擦除—回写”会有掉电丢失窗口。
- **记录格式**：使用魔数、格式版本、长度、序号、有效载荷和 CRC；不要仅判断第一个字不是 `0xFFFFFFFF`。
- **掉电保护**：常用两个独立擦除单元 A/B，先写并验证新记录，再写有效标记，确认成功后才回收旧区。启动时选择 CRC 正确且已提交的最新记录。
- **寿命**：查实际器件数据手册的擦写次数和保持条件；高频记录使用追加写、轮换扇区或其他存储介质。
- **结构体**：直接保存结构体会受填充字节、字节序和版本变化影响，长期格式宜明确序列化。

HAL 提供基础擦写能力，A/B 掉电保护、数据格式和磨损均衡需要由应用或存储中间件实现。

## 8. 擦写对系统运行的影响

F407 是单 Bank Flash。擦写期间访问同一 Flash Bank 可能使 CPU 等待，因此代码执行、中断响应、取常量和 HAL Tick 都可能受到影响。`_IT` 表示 HAL 的中断处理方式，不保证应用能在同 Bank 擦写期间照常运行。

不要用“整个擦除期间关闭中断”作为默认方案：这会影响通信、调度和依赖 Tick 的超时。若必须在擦写时维持实时响应，需要把相关执行路径、常量、向量表和中断依赖完整安排到 RAM，并核对芯片手册及勘误。

Flash 操作前要考虑供电稳定性、看门狗超时时间、通信缓存余量和掉电恢复。Flash 擦除不是毫秒任务中可以忽略的操作，其最大耗时应按数据手册评估。

F407 的 Flash 接口带指令/数据缓存；修改 Flash 内容后应按手册和 HAL 实现处理缓存一致性，避免读回旧值。这不同于 F7/H7 的 Cortex-M7 Cache，移植时须重新处理缓存和编程规则。
