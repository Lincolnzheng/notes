# FMC / FSMC 外部存储控制器

| 缩写            | 英文全称                              | 中文含义      |
| ------------- | --------------------------------- | --------- |
| `FSMC`        | Flexible Static Memory Controller | 灵活静态存储控制器 |
| `FMC`         | Flexible Memory Controller        | 灵活存储控制器   |
| `D0~D15`      | Data                              | 数据线       |
| `DB0~DB15`    | Data Bus                          | 数据总线      |
| `A0~Ax`       | Address                           | 地址线       |
| `CS`          | Chip Select                       | 片选        |
| `NE`          | Not Enable / Chip Enable          | 低有效片选/使能  |
| `NWE`         | Not Write Enable                  | 低有效写使能    |
| `NOE`         | Not Output Enable                 | 低有效输出/读使能 |
| `WR`          | Write                             | 写         |
| `RD`          | Read                              | 读         |
| `RS`          | Register Select                   | 寄存器选择     |
| `DC / D/C`    | Data / Command                    | 数据/命令选择   |
| `RESET / RST` | Reset                             | 复位        |
| `BL`          | Backlight                         | 背光        |
| `GPIO`        | General Purpose Input/Output      | 通用输入输出    |

## 1. FMC 是什么

**FMC（Flexible Memory Controller，灵活存储控制器）**用于把 CPU/DMA 发出的 AHB 访问转换为外部存储器或并行外设所需的地址、数据和控制时序。

配置完成后，外部器件会被映射到 STM32 的地址空间中，程序可以像读写普通内存一样访问它：

```text
CPU / DMA 读写某个映射地址
          ↓
     AHB 总线访问
          ↓
FMC 自动产生片选、地址、读写和数据时序
          ↓
外部 SRAM / NOR Flash / NAND Flash / SDRAM / LCD 控制器
```

因此，FMC 的核心作用是：

- 扩展片外存储器容量；
- 自动生成并行总线时序，减少 GPIO 模拟时序的工作量；
- 将带 8080/6800 并行接口的 LCD 控制器映射为“存储器”；
- 允许 CPU 或 DMA 直接访问外部器件。

> [!important] 先确认具体型号
> 并非所有 STM32 都有 FMC/FSMC，同一系列中不同封装引出的信号也可能不同。使用前必须查看具体芯片的数据手册、参考手册和引脚复用表。

## 2. FMC 与 FSMC 的区别

| 名称 | 英文 | 典型能力 | 常见型号示例 |
|---|---|---|---|
| FSMC | Flexible **Static** Memory Controller | SRAM、PSRAM、NOR Flash、NAND Flash、PC Card，以及 8080 类并行外设；通常不支持 SDRAM | STM32F103 大容量型号、STM32F407 |
| FMC | Flexible Memory Controller | 在静态存储器控制能力基础上，部分型号增加 SDRAM 等控制器 | STM32F429、部分 F7/H7 |

FMC 可以看作功能扩展后的 FSMC，但不同芯片的存储区、寄存器和 HAL 宏名称并不完全相同。F407 使用的是 **FSMC**；F429/439 使用 **FMC**，并增加了 SDRAM 控制器。

## 3. 支持的器件与接口

| 器件/接口 | 特点 | 典型用途 |
|---|---|---|
| SRAM | 异步静态存储器，无需刷新 | 扩展运行内存、帧缓冲区 |
| PSRAM/CRAM | 接口接近 SRAM，内部结构不同 | 大容量低成本存储 |
| NOR Flash | 可随机读取，适合存放代码和资源 | 字库、图片、外部程序存储 |
| NAND Flash | 容量大，按页/块操作，需要坏块和 ECC 管理 | 大容量数据存储 |
| SDRAM | 容量大，需要初始化与周期刷新 | 图形帧缓冲、大容量运行内存；仅支持 SDRAM 的 FMC 型号可用 |
| LCD 8080/6800 接口 | 本质上是并行寄存器接口，不是真正的存储器 | 驱动带控制器的 TFT LCD |

> [!note]
> FMC/FSMC 适合连接带 **MCU 8080/6800 并行接口**的 LCD 控制器。纯 RGB 屏需要 LTDC 或其他持续输出像素时序的控制器，MIPI 屏也不能直接由 FMC 驱动。

## 4. 常用信号

| 信号                    | 方向    | 作用                                       |
| --------------------- | ----- | ---------------------------------------- |
| `FMC_NEx`             | 输出    | 片选，低电平有效；不同 `NEx` 对应不同子 Bank             |
| `FMC_A[x]`            | 输出    | 地址线；连接 LCD 时可用其中一根作为 `RS/DC`             |
| `FMC_D[x]`            | 输入/输出 | 双向数据总线，常见宽度为 8、16 或 32 bit，具体取决于芯片和 Bank |
| `FMC_NOE`             | 输出    | 读使能/输出使能，低电平有效，常接 LCD 的 `RD`             |
| `FMC_NWE`             | 输出    | 写使能，低电平有效，常接 LCD 的 `WR`                  |
| `FMC_NBL[x]`          | 输出    | 字节通道选择/字节写使能                             |
| `FMC_NWAIT`           | 输入    | 外部器件请求延长访问周期                             |
| `FMC_NADV` / `FMC_NL` | 输出    | 地址与数据复用时的地址锁存信号                          |
| `FMC_CLK`             | 输出    | 同步突发访问时使用的时钟                             |

信号名前的 `N` 表示 **低电平有效**，例如 `NWE=0` 表示正在进行写操作。

## 5. 存储器映射

以下是 STM32F4 中常见的 FMC/FSMC 映射。是否真的存在某个 Bank，仍应以具体型号参考手册为准。

| 地址范围                        | Bank             | 典型用途                     |
| --------------------------- | ---------------- | ------------------------ |
| `0x6000_0000 ~ 0x6FFF_FFFF` | Bank 1，4 × 64 MB | NOR / PSRAM / SRAM / LCD |
| `0x7000_0000 ~ 0x7FFF_FFFF` | Bank 2           | NAND Flash               |
| `0x8000_0000 ~ 0x8FFF_FFFF` | Bank 3           | NAND Flash               |
| `0x9000_0000 ~ 0x9FFF_FFFF` | Bank 4           | PC Card（仅部分型号）           |
| `0xC000_0000 ~ 0xCFFF_FFFF` | SDRAM Bank 1     | SDRAM（仅支持 SDRAM 的 FMC）   |
| `0xD000_0000 ~ 0xDFFF_FFFF` | SDRAM Bank 2     | SDRAM（仅支持 SDRAM 的 FMC）   |

Bank 1 又分为四个 64 MB 子 Bank：

| 片选 | 基地址 |
|---|---|
| `NE1` | `0x6000_0000` |
| `NE2` | `0x6400_0000` |
| `NE3` | `0x6800_0000` |
| `NE4` | `0x6C00_0000` |

> [!warning]
> “映射空间大小”不等于外部芯片的真实容量。外部器件有多少地址线、实际容量是多少，仍由器件本身决定；访问超出真实容量的地址可能出现地址回卷或无效数据。

## 6. 异步访问时序

异步 SRAM、NOR Flash 和 LCD 8080 接口通常最关心以下参数：

| 参数 | 含义 |
|---|---|
| `ADDSET` / AddressSetupTime | 地址建立阶段持续时间 |
| `ADDHLD` / AddressHoldTime | 地址保持时间，主要在复用模式或特定访问模式中使用 |
| `DATAST` / DataSetupTime | 数据阶段持续时间，直接影响 `NOE/NWE` 有效脉宽 |
| `BUSTURN` / BusTurnAroundDuration | 连续访问之间的总线换向时间，避免读写方向切换时冲突 |
| `CLKDIV` | 同步访问时的输出时钟分频；异步访问通常不使用 |
| `DATLAT` | 同步突发访问的数据延迟；异步访问通常不使用 |
| `ACCMOD` | 访问模式 A/B/C/D，决定控制信号的具体波形 |

FMC 的异步时序通常以 `HCLK` 周期为单位：

$$
t_{HCLK}=\frac{1}{f_{HCLK}}
$$

计算时应从外部器件数据手册中找到地址建立时间、读访问时间、写脉宽、数据建立/保持时间和总线释放时间，再向上取整为足够的 HCLK 周期。

> [!warning] 不要直接照抄例程中的时序值
> `ADDSET=1、DATAST=5` 之类的值只对特定的 HCLK、PCB 负载和外部器件成立。首次调试可以适当放宽时序，通信稳定后再逐步缩短。

### 6.1 读写时序是否相同

- `ExtendedMode = DISABLE`：读和写共用一套时序；
- `ExtendedMode = ENABLE`：可分别配置读时序和写时序，HAL 初始化时通常分别传入 `Timing` 与 `ExtTiming`；
- 如果 LCD 的读时序明显慢于写时序，或者外部存储器对读写有不同要求，应考虑分开配置。

### 6.2 访问模式

模式 A/B/C/D 主要改变 `NEx`、`NOE`、`NWE`、地址和数据阶段之间的关系。常见的异步 SRAM 和 8080 LCD 多从模式 A 开始配置，但最终必须以外部器件的数据手册时序图和当前 STM32 参考手册为准。

## 7. 使用 FMC 驱动 8080 接口 LCD

### 7.1 LCD 接口分类

| 接口            | 是否通常带显存/控制器 | 驱动方式                        |
| ------------- | ----------- | --------------------------- |
| MCU 8080/6800 | 通常带 GRAM    | MCU 发送命令和显存数据，FMC/FSMC 可驱动  |
| RGB           | 通常不带完整帧缓冲   | 控制器必须持续输出像素和同步信号，通常使用 LTDC  |
| MIPI DSI      | 取决于模组       | 需要 MIPI DSI Host，FMC 不能直接代替 |

### 7.2 8080 接口连接

| LCD 信号             | FMC/FSMC 信号 | 说明                  |
| ------------------ | ----------- | ------------------- |
| `CS`               | `NEx`       | 片选                  |
| `RS` / `DC`        | 某根 `Ax`     | 选择命令寄存器或数据/显存       |
| `RD`               | `NOE`       | 读控制                 |
| `WR`               | `NWE`       | 写控制                 |
| `D0~D7` 或 `D0~D15` | 对应 `Dx`     | 8 bit 或 16 bit 数据总线 |
| `RESET`            | 普通 GPIO     | LCD 硬件复位，不一定经过 FMC  |
| 背光                 | GPIO 或 PWM  | 建议单独控制              |

FMC 将 LCD 看成一个异步 SRAM 类设备：访问一个地址表示写命令，访问另一个地址表示写/读数据。

### 7.3 命令地址与数据地址

假设：

- LCD 的 `RS/DC` 接到 FMC 的地址线 `Ax`；
- `RS/DC=0` 选择命令，`RS/DC=1` 选择数据；
- 子 Bank 基地址为 `BANK_BASE`。

则地址偏移为：

```text
8 bit 总线： DATA_ADDR = BANK_BASE + (1 << x)
16 bit 总线：DATA_ADDR = BANK_BASE + (1 << (x + 1))
```

16 bit 总线要多左移一位，是因为 Cortex-M 按字节编址，而 16 bit FMC 访问的最低地址位用于选择半字中的字节，外部 `A0` 对应内部地址位 `HADDR[1]`。

示例 1：`NE1 + A16 + 16 bit` 数据总线：

```c
#define LCD_CMD_ADDR   0x60000000UL
#define LCD_DATA_ADDR  0x60020000UL  // 0x60000000 + (1 << 17)

#define LCD_CMD   (*((volatile uint16_t *)LCD_CMD_ADDR))
#define LCD_DATA  (*((volatile uint16_t *)LCD_DATA_ADDR))
```

示例 2：`NE4 + A0 + 16 bit` 数据总线：

```text
命令地址：0x6C00_0000
数据地址：0x6C00_0002
```

> [!important]
> 地址必须根据实际的 `NEx`、`Ax` 和数据总线宽度计算，不能直接复制其他开发板的地址。

### 7.4 基本读写函数

```c
static inline void LCD_WriteCommand(uint16_t command)
{
    LCD_CMD = command;
}

static inline void LCD_WriteData(uint16_t data)
{
    LCD_DATA = data;
}

static inline uint16_t LCD_ReadData(void)
{
    return LCD_DATA;
}
```

必须使用 `volatile`，防止编译器删除或合并对外设映射地址的访问。

LCD 的完整启动通常为：

1. 初始化 FMC 和相关 GPIO；
2. 用普通 GPIO 复位 LCD 控制器；
3. 按屏厂提供的初始化序列写寄存器；
4. 设置显示方向、像素格式和显示窗口；
5. 写入 GRAM 完成画点、填充和图片显示；
6. 最后开启显示及背光。

## 8. CubeMX / HAL 配置思路

以异步 SRAM 或 8080 LCD 为例：

1. 在 CubeMX 中启用 FMC/FSMC 的 `NOR/SRAM` Bank；
2. 根据硬件选择 `NE1~NE4`、8/16 bit 数据宽度和所需地址线；
3. 配置为异步、地址数据不复用；LCD 通常可先按 `SRAM` 类型处理；
4. 根据外部器件数据手册配置 `ADDSET`、`DATAST`、`BUSTURN` 等参数；
5. CubeMX 配置 GPIO 复用和 FMC/FSMC 时钟；
6. HAL 调用 `HAL_SRAM_Init()` 完成控制器初始化；
7. 通过映射地址直接读写，或使用 `HAL_SRAM_Read_*()` / `HAL_SRAM_Write_*()`。

常见 HAL 调用关系：

```text
MX_FMC_Init()
    └─ HAL_SRAM_Init(&hsram, &Timing, &ExtTiming)
           └─ HAL_SRAM_MspInit()
                  ├─ 使能 FMC/FSMC 时钟
                  └─ 初始化地址线、数据线和控制线 GPIO
```

不同系列生成的类型名和宏可能是 `FMC_*` 或 `FSMC_*`，应以当前芯片的 HAL 头文件和 CubeMX 生成代码为准。

## 9. 外部 SRAM 的访问

初始化成功后可以通过指针直接访问：

```c
#define EXT_SRAM_BASE  0x60000000UL

volatile uint16_t *ext_sram = (volatile uint16_t *)EXT_SRAM_BASE;

ext_sram[0] = 0x1234;
uint16_t value = ext_sram[0];
```

调试时应先做简单的存储器测试：

1. 固定值测试：`0x0000`、`0xFFFF`、`0xAAAA`、`0x5555`；
2. 地址相关数据测试：向每个地址写入由地址计算出的值；
3. Walking 1/0 测试：检查数据线短路、开路或位序错误；
4. 扩大访问范围，检查地址线和容量；
5. 提高 FMC 时序速度后重复测试。

## 10. SDRAM 初始化概览

仅支持 SDRAM 的 FMC 型号可用。SDRAM 不能像异步 SRAM 一样只配置时序后立即访问，它通常需要以下命令序列：

1. 使能 FMC SDRAM 时钟；
2. 等待器件要求的上电稳定时间；
3. 发送 `Clock Enable` 命令；
4. 延时后发送 `Precharge All` 命令；
5. 发送若干次 `Auto Refresh`；
6. 加载 SDRAM Mode Register；
7. 设置刷新计数器；
8. 执行读写测试。

行数、列数、数据宽度、Bank 数、CAS Latency、刷新周期等参数必须来自 SDRAM 数据手册，不能只根据容量猜测。

## 11. 常见问题排查

| 现象 | 优先检查 |
|---|---|
| 完全无波形 | FMC 时钟、GPIO 复用、Bank 是否启用、访问地址是否正确 |
| 有片选但无读写脉冲 | `NOE/NWE` 引脚配置、读写权限、程序是否真的访问了 `volatile` 地址 |
| LCD 只接受命令，不接受数据 | `RS/DC` 所接 `Ax` 与数据地址偏移是否计算正确 |
| 颜色或数据位错乱 | 8/16 bit 总线宽度、数据线顺序、RGB565 高低字节顺序 |
| 低速正常、高速异常 | 放宽 `DATAST/ADDSET/BUSTURN`，检查走线、负载和电源完整性 |
| SRAM 部分地址重复 | 地址线缺失/接错、芯片真实容量与测试范围不符 |
| 读数据异常但写显示正常 | LCD 是否支持读操作、总线方向、`NOE`、读时序和虚读周期 |
| 出现 HardFault | 芯片是否真的具有该 FMC/FSMC Bank、时钟是否开启、映射地址是否合法 |

建议使用逻辑分析仪或示波器同时观察 `NEx`、`NOE/NWE`、`Ax` 和部分数据线，将实测波形与外部器件数据手册进行对照。

## 12. 关键结论

- FMC/FSMC 的本质是“**总线协议转换 + 地址映射**”；
- 配置后，软件通过指针读写映射地址，硬件自动产生外部时序；
- F407 是 FSMC，不能据此认为所有 F4 都支持 SDRAM；
- LCD 必须是带 8080/6800 MCU 并行接口的型号，纯 RGB/MIPI 接口不能直接套用；
- LCD 的命令/数据地址取决于 `NEx`、`RS/DC` 所接的 `Ax` 以及总线宽度；
- 所有时序值都应由 `HCLK` 和外部器件数据手册共同计算。

## 参考资料

- [RM0090：STM32F405/407/427/429 参考手册](https://www.st.com/resource/en/reference_manual/rm0090-stm32f407-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)
- [AN2790：使用 FSMC 连接 TFT LCD](https://www.st.com/resource/en/application_note/an2790-tft-lcd-interfacing-with-the-highdensity-stm32f10xxx-fsmc-stmicroelectronics.pdf)

## 相关笔记

- [[GPIO]]
- [[STM32系统框架]]
- [[HAL库]]
- [[TIMER]]
