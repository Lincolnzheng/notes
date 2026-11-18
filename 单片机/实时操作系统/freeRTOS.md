
# freeRTOS的一般写法
```c
// 文件：main.c

// @作用：LED1的任务函数
void vLED1Task(void *pvParameters)
{
    // 像在main()方法里写程序一样，写一个while循环
    while (1)
    {
        HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_SET); // 点亮
        vTaskDelay(pdMS_TO_TICKS(100)); // 延迟100ms

        HAL_GPIO_WritePin(LED1_GPIO_Port, LED1_Pin, GPIO_PIN_RESET); // 熄灭
        vTaskDelay(pdMS_TO_TICKS(100)); // 延迟100ms
    }

    vTaskDelete(NULL); // 一旦退出需删除自身
}


// @作用：LED3的任务函数
void vLED3Task(void *pvParameters)
{
    // 像在main()方法里写程序一样，写一个while循环
    while (1)
    {
        HAL_GPIO_WritePin(LED3_GPIO_Port, LED3_Pin, GPIO_PIN_SET); // 点亮
        vTaskDelay(pdMS_TO_TICKS(30)); // 延迟30ms

        HAL_GPIO_WritePin(LED3_GPIO_Port, LED3_Pin, GPIO_PIN_RESET); // 熄灭
        vTaskDelay(pdMS_TO_TICKS(150)); // 延迟150ms
    }

    vTaskDelete(NULL); // 一旦退出需删除自身
}


int main(void)
{
    .........

    // 创建LED1任务，其实就是把vLED1Task()方法交给调度器去执行
    xTaskCreate(vLED1Task, "LED1", 128, NULL, 1, NULL);

    // 创建LED3任务，其实就是把vLED3Task()方法交给调度器去执行
    xTaskCreate(vLED3Task, "LED3", 128, NULL, 1, NULL);

    // 启动调度器
    vTaskStartScheduler();

    // 调度器永不退出，除非因创建空闲任务或者定时器任务失败
    // 因此只要程序运行正常，代码就不会执行到这里！
    while (1)
    {
    }
}
```


## 内核源文件选择

| 文件名               | 必选  | 作用                                                                          | 关联的内核对象或功能                       |
| ----------------- | :-: | --------------------------------------------------------------------------- | -------------------------------- |
| `tasks.c`          |  ✅  | 与任务调度相关的内容存储在此文件中，该文件必须添加到工程中                                               | 任务、任务通知、调度器                      |
| `list.c`          |  ✅  | 用于实现“列表”数据类型，仅 FreeRTOS 内部使用，但它是支撑 FreeRTOS 运行的关键。该文件必须添加到工程中               | —                                |
| `queue.c`         |     | 与队列、各种信号量相关的源文件。如果项目中用到队列或信号量，则需要添加此文件                                      | 队列、各种信号量（二进制信号量、计数信号量、互斥锁、递归互斥锁） |
| `timers.c`         |     | 与软件定时器相关的源文件。如果项目中用到了定时器，则需要添加此文件                                           | 软件定时器                            |
| `event_groups.c`   |     | 与事件组相关的源文件。如果项目中使用到了事件组，则需要添加此文件                                            | 事件组                              |
| `croutine.c`      |     | 与协程相关的源文件。协程是一种比较早期的技术，用于资源极度匮乏的嵌入式硬件系统，现在已经废弃。                             | 协程                               |
| `stream_buffer.c` |     | 与流缓冲区和消息缓冲区有关的源文件。如果项目中使用到了流缓冲区或者消息缓冲区，则需要添加此文件。注意，本课程暂不介绍与流缓冲区和消息缓冲区相关的内容。 | 流缓冲区、消息缓冲区                       |


## 需要修改的配置项
```c
#define configCPU_CLOCK_HZ    ( ( unsigned long ) 168000000 )
#define configTICK_TYPE_WIDTH_IN_BITS              TICK_TYPE_WIDTH_32_BITS
#define configKERNEL_INTERRUPT_PRIORITY          (15 << 4)
#define configMAX_SYSCALL_INTERRUPT_PRIORITY     (5 << 4)
#define configUSE_TIME_SLICING                     1
#define configCHECK_FOR_STACK_OVERFLOW        0
```

# FreeRTOS 内核移植总结

来源：[第二章 FreeRTOS内核移植](https://kcn06ia36zs4.feishu.cn/wiki/AJd5wwaVWiKFERkFxxncsLu0nDb)，重点整理第 2.2～2.4 节的移植流程。教程示例环境为 **STM32F103RCT6（Cortex-M3）+ STM32CubeIDE（GCC）**。下面的“补充说明”用于解释现有笔记和容易混淆的地方。

## 1. 移植到底在做什么

把 FreeRTOS 内核源码加入自己的工程，选择与工具链、处理器内核匹配的移植层，再配置运行参数和中断入口，让调度器能够在目标芯片上运行。

记忆顺序：**源码 → 头文件 → 移植层 → 内存管理 → 配置文件 → 中断 → HAL 时基 → 任务测试**。

## 2. 添加源码与头文件

教程下载并解压 FreeRTOS LTS 源码包，主要使用其中的 `FreeRTOS/FreeRTOS-Kernel`。版本不同，压缩包名称和目录位置可能变化。

在工程根目录建立：

```text
FreeRTOS/
├── Inc/                 内核头文件、portmacro.h、FreeRTOSConfig.h
└── Src/                 内核 .c 文件、port.c、一个 heap_x.c
```

- 把内核根目录下需要的 `.c` 文件加入 `Src`，具体作用见上面的源文件表。教程为了方便，先复制全部内核 `.c` 文件。
- 把内核 `include` 目录里的 `.h` 文件复制到 `Inc`，并将 `Inc` 加入编译器头文件搜索路径。
- 使用任务与调度器时包含 `FreeRTOS.h` 和 `task.h`；使用其他功能时再包含对应头文件。
- **复制文件和参与编译是两件事**：确认新增 `.c` 文件实际加入构建，且没有重复编译。

## 3. 选择移植层和内存管理方案

移植层目录按下面的组合选择：

```text
portable/<工具链>/<处理器内核>/
```

教程使用 GCC 编译器和 Cortex-M3 内核，因此选 `portable/GCC/ARM_CM3`，把 `port.c` 加入 `Src`，把 `portmacro.h` 加入 `Inc`。换芯片或编译器时必须重新核对，不能仅按“都是 STM32”选择。

动态创建任务等内核对象需要堆内存管理。`portable/MemMang` 提供 `heap_1.c`～`heap_5.c`，**只选择一种方案**。教程选用 `heap_4.c`，将它加入 `Src`。

补充说明：使用 `heap_4` 时，需要根据任务栈、任务控制块及其他动态对象的总需求设置 `configTOTAL_HEAP_SIZE`。这是 FreeRTOS 管理的堆，任务创建失败时应检查堆空间和返回值。

## 4. 配置 FreeRTOSConfig.h

教程从内核示例目录复制 `FreeRTOSConfig.h` 到工程，再按实际硬件修改。它通过宏定义控制内核的参数和功能，示例值不能直接照搬。

| 配置项 | 含义与移植要点 |
| --- | --- |
| `configCPU_CLOCK_HZ` | 填实际 CPU 内核时钟。教程 STM32F103 使用 HCLK：默认 8 MHz 填 `8000000`，改为 72 MHz 后填 `72000000`。本笔记原有的 `168000000` 仅在实际内核时钟为 168 MHz 时成立。 |
| `configTICK_TYPE_WIDTH_IN_BITS` | 决定保存系统节拍的 `TickType_t` 位宽；本笔记使用 `TICK_TYPE_WIDTH_32_BITS`。 |
| `configKERNEL_INTERRUPT_PRIORITY` | 设置内核中断优先级；本笔记示例为 `(15 << 4)`。 |
| `configMAX_SYSCALL_INTERRUPT_PRIORITY` | 限定允许调用内核中断接口的中断优先级范围；教程示例为 `(5 << 4)`。 |
| `configUSE_TIME_SLICING` | 设置是否使用时间片；设为 `1` 启用时间片。 |
| `configCHECK_FOR_STACK_OVERFLOW` | 本笔记原有值为 `0`，表示关闭栈溢出检查；这是配置选择，并非移植必须关闭的功能。 |

补充说明：

- **tick 是节拍数，不一定是毫秒数**。`configTICK_RATE_HZ` 决定每秒的 tick 数；任务延时使用 `pdMS_TO_TICKS(ms)` 转换。HAL 的时间计数与 FreeRTOS 的 tick 计数是两套计数。
- 抢占由 `configUSE_PREEMPTION` 控制；`configUSE_TIME_SLICING` 主要影响同优先级就绪任务的时间片轮转，不能用它代替抢占开关。
- Cortex-M 中断优先级**数值越小，紧急程度越高**。若实现了 4 个优先级位，逻辑数值 0～15 写入寄存器时左移 4 位，因此有 `15 << 4`、`5 << 4` 的写法。
- 对上述配置，逻辑优先级数值 5～15 的中断可以使用适用的 `...FromISR` API；数值 0～4 的中断不能调用 FreeRTOS API。任务优先级与 [[NVIC|中断优先级]] 不能混为一谈。
- HAL 的 `HAL_NVIC_SetPriority()` 通常传未移位的逻辑优先级，例如 `5`，不要直接把 `(5 << 4)` 当作它的参数。

## 5. 安装三个内核中断处理函数

FreeRTOS 的 Cortex-M 移植依赖下面三个中断，处理函数由 `port.c` 提供：

| 中断 | FreeRTOS 处理函数 | 作用 |
| --- | --- | --- |
| SVC | `vPortSVCHandler` | 启动第一个任务。 |
| PendSV | `xPortPendSVHandler` | 保存、恢复任务上下文，完成任务切换。 |
| SysTick | `xPortSysTickHandler` | 维护系统节拍，驱动延时和调度。 |

教程直接修改启动文件 `Core/Startup/startup_stm32f103rctx.s` 中的向量表，把原来的三个入口替换为对应 FreeRTOS 函数：

```text
SVC_Handler      → vPortSVCHandler
PendSV_Handler   → xPortPendSVHandler
SysTick_Handler  → xPortSysTickHandler
```

注意函数名大小写，以实际 `port.c` 为准。关键是让中断向量最终指向正确的处理函数，避免仍然进入空处理函数或旧 HAL 的 SysTick 处理函数。启动文件的具体行号随工程变化，不应照抄教程行号。

## 6. 为 HAL 另选时间基准

按照教程直接把 SysTick 向量交给 FreeRTOS 后，原来 HAL 在 `SysTick_Handler` 中调用 `HAL_IncTick()` 的路径就不会执行。HAL 时间计数停止，会影响 `HAL_GetTick()`、`HAL_Delay()` 和依赖它的超时判断。

教程在 CubeIDE 图形配置中，将 **System Core → SYS → Timebase Source（时间基准）改为 TIM6**，重新生成代码，让其他定时器维护 HAL 的毫秒计数。具体使用哪个定时器取决于芯片资源和工程安排。

任务中需要等待时，优先使用 `vTaskDelay(pdMS_TO_TICKS(...))`，让任务进入阻塞状态；它和 `HAL_Delay()` 的作用方式不同。

## 7. 最小运行验证

教程最后通过两个不同闪烁周期的 LED 任务验证移植。本笔记开头已有两任务示例，可沿用它检查任务创建、调度和延时是否正常。

- [ ] 内核源码、正确的 `port.c` 和一种 `heap_x.c` 已参与编译。
- [ ] 头文件搜索路径包含内核头文件、`portmacro.h` 和 `FreeRTOSConfig.h`。
- [ ] CPU 时钟、tick 配置和中断优先级与目标芯片一致。
- [ ] SVC、PendSV、SysTick 的向量入口正确。
- [ ] HAL 时基已按教程改到其他定时器；调试接口已启用。
- [ ] 检查 `xTaskCreate()` 返回值是否为 `pdPASS`，然后启动调度器。
- [ ] 两个 LED 都按各自设置的节奏运行，说明任务调度和阻塞延时的基本链路正常。

补充说明：最小任务测试通过只是初步验证。如果 `vTaskStartScheduler()` 返回，优先检查创建空闲任务、定时器任务所需的内存；如果任务能启动但延时不恢复，检查 SysTick 入口与节拍配置；如果 HAL 延时或超时失效，检查 HAL 时基。


# 五种堆内存管理方案
pvPortMalloc()
vPortFree()

heap_1 极其简单只能分配，不能释放。
heap_2 