# STM32 低功耗：进入与退出 API

本笔记以 **STM32F407 + STM32F4 HAL** 为主。优先记住每个模式的进入调用、唤醒条件和恢复调用；其他系列的 Stop 0/1/2、Shutdown 等接口不能直接照搬。

## 进入／退出速查（重点）

| 模式          | **进入 API**                                                           | **退出方式**               | **唤醒后的恢复接口**                                                                         |
| ----------- | -------------------------------------------------------------------- | ---------------------- | ------------------------------------------------------------------------------------ |
| **Sleep**   | `HAL_PWR_EnterSLEEPMode(PWR_MAINREGULATOR_ON, PWR_SLEEPENTRY_WFI)`   | 中断唤醒；WFE 入口则由事件唤醒      | 若暂停了 Tick：`HAL_ResumeTick()`；通常无需重配时钟                                                |
| **Stop**    | `HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI)` | 已配置的 EXTI 中断／事件唤醒      | `SystemClock_Config()` 恢复系统时钟；若暂停了 Tick，还需 `HAL_ResumeTick()`                        |
| **Standby** | `HAL_PWR_EnterSTANDBYMode()`                                         | WKUP、RTC 等唤醒，按复位流程重新启动 | 启动时用 `__HAL_PWR_GET_FLAG(PWR_FLAG_SB)` 判断，再用 `__HAL_PWR_CLEAR_FLAG(PWR_FLAG_SB)` 清标志 |

> [!important] 退出接口怎么记
> **三个模式都没有独立的 `HAL_PWR_Exit…Mode()` 接口。** 退出由唤醒源触发。
> **Sleep：醒来继续；Stop：醒来恢复时钟；Standby：醒来重新启动。**
> `HAL_ResumeTick()` 恢复 HAL 时间基准，`SystemClock_Config()` 是项目的时钟配置函数；它们都在唤醒后执行，不负责触发唤醒。
> 依据：[ST STM32F4 HAL PWR 实现](https://github.com/STMicroelectronics/stm32f4xx-hal-driver/blob/master/Src/stm32f4xx_hal_pwr.c)。

使用 Stop、Standby 的 PWR 配置前，使能 PWR 时钟（项目初始化中已完成时无需重复）：

```c
__HAL_RCC_PWR_CLK_ENABLE();
```

## F407 的主要运行状态

| 模式      | CPU             | 主时钟与外设                | SRAM/寄存器              | 典型唤醒方式               | 唤醒后执行位置   |
| ------- | --------------- | --------------------- | --------------------- | -------------------- | --------- |
| Run     | 运行              | 按配置运行                 | 保持                    | —                    | 继续运行      |
| Sleep   | CPU 时钟停止        | 其他时钟可继续               | 保持                    | NVIC 中断或事件           | 从休眠调用后继续  |
| Stop    | 1.2 V 域时钟停止，内核停 | HSI/HSE/PLL 停止，大部分外设停 | 保持                    | 已配置的 EXTI 中断/事件      | 从休眠调用后继续  |
| Standby | 1.2 V 域断电       | 大部分资源关闭               | SRAM 和大部分寄存器丢失；备份域可保持 | WKUP、RTC、NRST、IWDG 等 | 按复位流程重新启动 |

从 Sleep → Stop → Standby，通常功耗逐步降低，但可运行资源变少、唤醒条件更受限，恢复工作也更多。实际电流和唤醒时间应查具体型号数据手册，不使用单一经验值代替。

## Sleep 模式

### 进入 API（重点）

```c
HAL_PWR_EnterSLEEPMode(PWR_MAINREGULATOR_ON, PWR_SLEEPENTRY_WFI);
```

### 退出／恢复接口（重点）

**没有主动退出 API。** 已配置的中断到来后唤醒，入口函数返回；如果进入前调用了 `HAL_SuspendTick()`，返回后调用 `HAL_ResumeTick()`。系统时钟配置保持。

Sleep 只停止 Cortex-M4 内核时钟，外设可继续运行并通过中断唤醒 CPU。它适合等待 UART、定时器、DMA、ADC 等外设完成。

### 特点

- SRAM、寄存器和系统时钟配置保持。
- 唤醒速度快，通常不需要重新配置系统时钟。
- 若 SysTick 仍周期中断，CPU 会频繁唤醒，难以获得持续低功耗。
- 没有使用的外设时钟仍会消耗功耗，应按需关闭。

### HAL 示例

```c
/**
 * @brief 使用 WFI 进入 Sleep，并仅由已配置的业务中断唤醒。
 * @note 若系统或 RTOS 依赖 SysTick 调度，不可直接照搬暂停 Tick 的处理。
 */
static void EnterSleepMode(void)
{
    HAL_SuspendTick();
    HAL_PWR_EnterSLEEPMode(PWR_MAINREGULATOR_ON, PWR_SLEEPENTRY_WFI);
    HAL_ResumeTick();
}
```

`HAL_PWR_EnterSLEEPMode()` 返回时，CPU 已经被某个中断或事件唤醒。应在对应中断里清除硬件标志，避免再次进入后立即唤醒。

## Stop 模式

### 进入 API（重点）

```c
HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON, PWR_STOPENTRY_WFI);
```

第一个参数也可用 `PWR_MAINREGULATOR_ON`；第二个参数可选 `PWR_STOPENTRY_WFE`，按事件唤醒。

### 退出／恢复接口（重点）

**没有主动退出 API。** EXTI 唤醒后入口函数返回，再执行项目的 `SystemClock_Config()` 恢复原系统时钟。若暂停过 HAL Tick，需恢复 `HAL_ResumeTick()`。

`SystemClock_Config()` 是项目函数，不是 HAL 的退出模式接口；其内部通常调用 RCC HAL 接口恢复 HSE／PLL 和总线时钟。

Stop 模式关闭 1.2 V 域时钟并停止 HSI、HSE 和 PLL，同时保持 SRAM 和大部分寄存器内容。F407 可选择主稳压器或低功耗稳压器：

| 参数                         | 特点           |
| -------------------------- | ------------ |
| `PWR_MAINREGULATOR_ON`     | 恢复较快，功耗相对高   |
| `PWR_LOWPOWERREGULATOR_ON` | 功耗更低，恢复时间相对长 |

### 可用唤醒源

Stop 由已配置为中断或事件的 EXTI 线唤醒，包括外部 GPIO EXTI 和映射到 EXTI 的 RTC、PVD 等内部事件。只配置 GPIO 输入但不配置 EXTI，不能唤醒 Stop。

进入前必须确认：

- 唤醒源在 Stop 时仍能工作。
- EXTI 边沿和 NVIC 配置正确。
- 旧的外设/EXTI/NVIC 标志已经清除。
- 唤醒信号不会一直保持在导致重复触发的有效状态。

### 唤醒后为什么要恢复系统时钟


F407 从 Stop 唤醒后，HSI 被打开并作为 [[时钟树|系统时钟]]；进入前使用的 HSE/PLL 配置不会自动恢复为原运行状态。因此通常要再次调用项目的 `SystemClock_Config()`，再使用依赖精确时钟的 UART、定时器等外设。

### HAL 示例

```c
/**
 * @brief 进入低功耗稳压器 Stop 模式，并在唤醒后恢复系统时钟。
 * @note 唤醒 EXTI/RTC 必须在调用本函数前配置并清除旧标志。
 */
static void EnterStopMode(void)
{
    HAL_SuspendTick(); // 暂停 HAL Tick 中断，不是关闭整个 SysTick

    HAL_PWR_EnterSTOPMode(PWR_LOWPOWERREGULATOR_ON,
                          PWR_STOPENTRY_WFI);

    /* 先恢复 Tick，让时钟配置中的 HAL 超时等待能够推进。 */
    HAL_ResumeTick();
    SystemClock_Config(); // 恢复 HSE/PLL、总线时钟及相应时间基准
}
```

如果外设寄存器虽然保留，但其时钟源、DMA 或通信对端状态已经变化，还需要在项目层恢复或重新启动外设，不能只恢复 RCC。
> [!note] 本示例的 Tick 顺序
> 默认 HAL 时间基准为 SysTick，时钟配置中的超时等待依赖 `HAL_GetTick()`，所以示例先恢复 Tick 再重配时钟；重配前 Tick 周期可能暂时不准确。使用自定义时间基准或 RTOS 时，应按项目的时钟恢复流程处理。
> `HAL_SuspendTick()` / `HAL_ResumeTick()` 在默认实现中只禁用／使能 SysTick 中断。[ST HAL 时间基准实现](https://github.com/STMicroelectronics/stm32f4xx-hal-driver/blob/master/Src/stm32f4xx_hal.c)

## Standby 模式

### 进入 API（重点）

```c
HAL_PWR_EnterSTANDBYMode();
```

### 退出／启动处理接口（重点）

**没有主动退出 API，也不会从入口函数后继续。** 唤醒后重新执行启动流程，在启动阶段查询和清除 Standby 标志：

```c
if (__HAL_PWR_GET_FLAG(PWR_FLAG_SB) != RESET)
{
    __HAL_PWR_CLEAR_FLAG(PWR_FLAG_SB);
    /* 在这里处理 Standby 唤醒后的业务。 */
}
```

WKUP 唤醒源在进入前用 `HAL_PWR_EnableWakeUpPin(PWR_WAKEUP_PIN1)` 使能。`HAL_PWR_DisableWakeUpPin()` 只是禁用该唤醒源，不是退出 Standby。

Standby 关闭主稳压器并让 1.2 V 域断电，功耗最低，但 SRAM、CPU 上下文和大部分寄存器都会丢失。RTC 和备份寄存器可以由备份域/VBAT 保持。

### F407 常见唤醒条件

- WKUP 引脚上升沿。
- RTC Alarm A / Alarm B。
- RTC Wake-up Timer。
- RTC Tamper 或 TimeStamp 事件。
- NRST 外部复位。
- IWDG 复位。

Standby 唤醒后按复位流程从复位向量重新执行，不会从 `HAL_PWR_EnterSTANDBYMode()` 后一行正常继续。软件应在启动阶段读取 `PWR_FLAG_SB`，判断这次启动是否来自 Standby。

### 进入前的标志处理

WKUP 引脚已经为高时，如果直接使能，可能立即产生唤醒事件。通常应先确保引脚处于无效电平，再清除 WU 标志并使能唤醒引脚。

```c
/**
 * @brief 配置 WKUP1 并进入 Standby。
 * @note 正常进入后本函数不会返回；WKUP 引脚必须先处于低电平。
 */
static void EnterStandbyMode(void)
{
    __HAL_PWR_CLEAR_FLAG(PWR_FLAG_WU);
    HAL_PWR_EnableWakeUpPin(PWR_WAKEUP_PIN1);
    HAL_PWR_EnterSTANDBYMode();
}
```

### 启动时识别 Standby 唤醒

```c
/**
 * @brief 在系统启动阶段检查并清除 Standby 标志。
 * @return 来自 Standby 唤醒返回 1，否则返回 0。
 */
static int WasStandbyWakeup(void)
{
    if (__HAL_PWR_GET_FLAG(PWR_FLAG_SB))
    {
        __HAL_PWR_CLEAR_FLAG(PWR_FLAG_SB);
        return 1;
    }
    return 0;
}
```

需要跨 Standby 保存少量状态时，可使用 RTC 备份寄存器，并配合魔数、版本号和 CRC 判断数据是否有效。备份寄存器不是无限可靠的文件系统，也要考虑 VBAT 丢失和备份域复位。

## 常用 HAL 接口

| 接口 | 作用 |
|---|---|
| `__HAL_RCC_PWR_CLK_ENABLE()` | 使能 PWR 外设时钟 |
| `HAL_PWR_EnterSLEEPMode()` | 进入 Sleep |
| `HAL_PWR_EnterSTOPMode()` | 进入 Stop |
| `HAL_PWR_EnterSTANDBYMode()` | 进入 Standby |
| `HAL_PWR_EnableWakeUpPin()` | 使能 Standby WKUP 引脚 |
| `HAL_PWR_DisableWakeUpPin()` | 禁用 WKUP 引脚 |
| `__HAL_PWR_GET_FLAG()` | 查询 WU、SB 等 PWR 标志 |
| `__HAL_PWR_CLEAR_FLAG()` | 清除 WU、SB 等 PWR 标志 |
| `HAL_PWR_EnableSleepOnExit()` | 从 ISR 返回后自动进入 Sleep |
| `HAL_SuspendTick()` / `HAL_ResumeTick()` | 暂停/恢复 HAL SysTick 中断 |

原笔记中的 `__HAL_RCC_PWR_CLK_EANBLE()` 是拼写错误，正确宏为：

```c
__HAL_RCC_PWR_CLK_ENABLE();
```

## WFI、WFE 与 Sleep-on-exit

### WFI：Wait For Interrupt

`WFI` 等待中断。使用 HAL 时常见参数为：

```c
PWR_SLEEPENTRY_WFI
PWR_STOPENTRY_WFI
```

如果已有满足条件的中断处于挂起状态，CPU 可能不会真正睡眠或会立即唤醒。因此进入前要清除唤醒外设的旧标志和 NVIC Pending 状态。

### WFE：Wait For Event

`WFE` 等待事件，事件不一定进入中断服务函数。事件寄存器中残留的旧事件可能导致立即返回；使用前要理解 `SEV/WFE` 清除序列及当前 HAL 版本的 WFE 入口实现。

### Sleep-on-exit

使能 Sleep-on-exit 后，CPU 从中断退出时会再次进入 Sleep，适合“全部工作都由中断驱动”的结构：

```c
HAL_PWR_EnableSleepOnExit();
HAL_PWR_DisableSleepOnExit();
```

如果主循环还承担协议解析或状态机推进，不应直接启用，否则主循环可能得不到运行机会。

> [!tip] Sleep-on-exit 的退出控制
> 在唤醒 ISR 中调用 `HAL_PWR_DisableSleepOnExit()`，可取消中断返回后自动再次睡眠，使主循环继续执行。它控制自动再入睡，不是普通 Sleep 模式的唤醒接口。

## RTC 定时唤醒


需要周期性采集时，可由 [[RTC(Real Time Clock)|RTC]] 的 Wake-up Timer 或 Alarm 唤醒 Stop/Standby：

```text
配置 RTC 时钟源与唤醒中断
        ↓
清除 RTC、EXTI、PWR 的旧标志
        ↓
设置下一次 Wake-up / Alarm
        ↓
停止不需要的外设并进入低功耗
        ↓
RTC 事件产生 → 唤醒
        ↓
清除标志 → 恢复时钟/外设 → 执行业务
```

RTC 使用 LSE 时通常精度更高；LSI 成本低但误差较大，并受温度、电压和个体差异影响。具体中断回调和清标志顺序应按照所用 STM32 系列的 RTC HAL 实现。

## 进入低功耗的完整流程

### 进入前

1. 确认系统没有必须立即处理的任务。
2. 选择允许的唤醒源和目标最长唤醒时间。
3. 停止新的 DMA、通信和 Flash 写入请求；等待已开始的关键操作安全结束。
4. 让传感器、RS-485 收发器、屏幕等外部器件进入合适状态。
5. 保存需要保留的软件状态。
6. 配置唤醒源，清除外设、EXTI、NVIC 和 PWR 的旧标志。
7. 必要时暂停 SysTick，再调用低功耗入口。

### 唤醒后

1. 判断唤醒原因并清除对应标志。
2. Stop 唤醒后恢复系统时钟。
3. 恢复 GPIO、外设、DMA 和通信状态。
4. 恢复 SysTick 或 RTOS 调度。
5. 检查休眠期间经过的时间、超时和协议状态。
6. Standby 唤醒按一次新的启动处理，只从备份域恢复必要信息。

> [!warning] 竞态
> “检查任务为空”和执行 WFI 之间若刚好发生中断，程序可能处理完中断后才进入睡眠，导致错过预期唤醒。事件驱动系统需要设计无竞态的空闲入口；RTOS 应优先使用其 Tickless Idle/低功耗钩子，不在任务中随意暂停系统 Tick。

## 降低功耗的基本思路

动态功耗可粗略理解为：

`Pdynamic ∝ C × V² × f × 活动率`

因此可以从以下方向降低功耗：

- 降低系统和外设时钟频率。
- 关闭暂时不用的外设时钟。
- 让 CPU 使用 `WFI` / `WFE` 等待，不做空循环轮询。
- 进入 Sleep、Stop 或 Standby，逐步关闭更多时钟和电源域。
- 降低 GPIO 翻转、悬空输入和外部上拉/下拉产生的电流。
- 关闭或休眠传感器、收发器、显示屏等板外器件。

> [!important]
> MCU 数据手册中的典型低功耗电流只描述规定测试条件下的芯片本身。开发板上的电源指示灯、ST-LINK、稳压器静态电流和外部器件，往往比 MCU 更耗电。

## GPIO 与外设处理

### 未使用 GPIO

未连接引脚若保持数字浮空输入，可能因噪声反复翻转输入级而增加功耗。常见做法是配置为模拟模式、无上下拉；但必须排除调试口、晶振、Boot、唤醒脚以及连接外设的引脚。

### 外部器件反向供电

MCU 掉电或把外设电源关闭后，如果 MCU GPIO 仍输出高电平，电流可能经外设保护二极管反向灌入，导致功耗异常甚至损坏。应依据外设手册把引脚置为安全电平/高阻，或增加电源域隔离。

### 通信与 DMA

进入低功耗前，检查 [[DMA]] 的传输完成状态，确认缓冲区已停止使用。

- 进入 Stop 前确认 UART/SPI/I²C 是否正在发送，不能截断协议帧。
- Stop 后外设时钟通常停止，不能假定 DMA 继续工作。
- Sleep 中外设和 DMA 可以继续工作，完成中断可唤醒 CPU。
- 唤醒后重新确认外设句柄、DMA 状态和通信对端是否仍同步。

## 调试与功耗测量

调试器可能配置 DBGMCU，使内核或外设在 Sleep/Stop/Standby 下保留调试能力，从而显著增加电流。正式测量前应确认调试低功耗设置，并尽量断开 ST-LINK。

测量时建议：

1. 先测整板，再分离 MCU 与板外器件贡献。
2. 去除或计入电源 LED、USB-UART、调试器和稳压器静态电流。
3. 使用有足够动态范围的电流表/功耗分析仪，注意串联表的负担电压。
4. 同时观察唤醒周期，平均电流取决于运行电流、睡眠电流和占空比。
5. 用 GPIO 翻转标记进入/退出区间，配合示波器或逻辑分析仪检查实际睡眠时间。

平均电流可近似计算：

`Iavg = (Irun × trun + Isleep × tsleep) / (trun + tsleep)`

降低唤醒次数、缩短运行时间有时比追求更深的休眠模式更有效。



## 参考资料

- [ST RM0090：STM32F405/407/427/429 参考手册](https://www.st.com/resource/en/reference_manual/dm00031020-stm32f405415xx-stm32f407417xx-stm32f427437xx-and-stm32f429439xx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf)
- [ST AN4365：Using STM32F4 MCU power modes with best dynamic efficiency](https://www.st.com/resource/en/application_note/dm00096220-using-stm32f4-mcu-power-modes-with-best-dynamic-efficiency-stmicroelectronics.pdf)
- [ST STM32F4 HAL PWR 头文件](https://github.com/STMicroelectronics/stm32f4xx-hal-driver/blob/master/Inc/stm32f4xx_hal_pwr.h)
