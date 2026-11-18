# CMSIS简介
CMSIS (微控制器软件接口标准)：Cortex Microcontroller Software Interface Standard，是由ARM和与其合作的芯片厂商、软件工具厂商，共同制定的标准
![[Pasted image 20260730173309.png]]
内核的抽象层，提供内核的API来调用内核的功能。
# HAL简介
标准外设库 (Standard Peripheral Libraries)
HAL库(硬件抽象层)：Hardware Abstraction Layer
LL库：Low Layer
这三个库是外设控制的函数，来定义外设的行为，使用外设

从标准外设库到HAL库再到LL库，逐渐底层，逐渐变高效，HAL库和LL库是一同发布配合使用的，标准外设库现在已经停止维护，逐渐淘汰了。

| 文件名称                                         | 描述                                                   |
| -------------------------------------------- | ---------------------------------------------------- |
| stm32f1xx.h                                  | 是所有F1系列的顶层头文件，通过条件编译来包含某个芯片的头文件，定义通用的枚举类型，定义通用的宏定义   |
| stm32f103xe.h                                | 包含：中断编号定义、外设寄存器结构体类型定义、寄存器映射、寄存器位定义、外设判定             |
| system_stm32f1xx.c<br><br>system_stm32f1xx.h | 定义了系统初始化函数 SystemInit和系统时钟更新函数 SystemCoreClockUpdate |
| startup_stm32f103xe.s                        | 大容量F103系列芯片的启动文件                                     |
# HAL库框架结构



HAL库文件介绍

| 文件                                               | 描述                                                                 |
| ------------------------------------------------ | ------------------------------------------------------------------ |
| sm32f1xx_hal.c<br>stm32f1xx_hal.h                | HAL库初始化、系统滴答、HAL库延时等相关函数                                           |
| stm32f1xx_hal_conf.h                             | HAL库的用户配置文件，用于裁剪HAL库、配置晶振参数等                                       |
| stm32hxx_hal_def.h                               | 包含HAL库通用的枚举类型数据和宏定义                                                |
| stm32f1xx_hal_cortex.h<br>stm32f1xx_hal_cortex.c | 内核通用函数定义和声明，如NVIC、MPU、系统软复位、Systick等，其实主要是对core_cm3.h 文件的相关函数再次封装。 |
| stm32f1xx_hal_ppp.c<br>stm32f1xx_hal_ppp.h       | 某任意外设驱动源码，PPP表示任意外设                                                |
| stm32f1xx_hal_ppp_ex.c<br>stm32f1xx_hal_ppp_ex.h | 主要是存放外设的扩展（特殊）功能的驱动源码，PPP表示任意外设                                    |
| stm32f1xx_II_ppp.c<br>stm32f1xx_II_ppp.h         | LL库驱动源码，在部分STM32F1xx_hal_ppp.c 或stm32f1xx_hal_ppp_ex.c中会被调用        |
**例子：**
初始化/反初始化函数:HAL_PPP_Init(), HAL_PPP_DeInit()
外设读写函数:HAL_PPP_Read(),HAL_PPP_Write(),HAL_PPP_Transmit(), HAL_PPP_Receive()
控制函数:HAL_PPP_Set (),HAL_PPP_Get ()
状态和错误:HAL_PPP_GetState (), HAL_PPP_GetError ()


# HLA库回调函数：
HAL_PPP_MspInit()/ _ MspdeInit()：[[C语言补充|弱定义函数]]（_ weak),空函数，允许重新定义用来初始化多个外设

HAL_PPP_Init()----->HAL_PPP_MspInit()
		调用MSP回调函数
![[Pasted image 20260806120309.png]]
`_MspInit` 就是把"开时钟 + 配中断"从你的业务代码里剥离出来，放到 HAL 规定的回调位置。你手动做了它的事，就不需要写它；你偷懒没做，它就必须存在。

HAL_PPP_ProcessCpltCallback举例: HAL_USART_TxCpltCallback由外设中断或DMA中断调用，HAL库中断公共处理函数已经实现对中断标记位读取、判断和清除操作，**用户只需要专注于中断逻辑功能的实现即可**

HAL_PPP_ErrorCallback :外设或DMA中断中发生的错误，用于发生错误处理

在hal_config里来裁剪HAL库,或者把 .c文件不包括到文件中

HSE(High-Speed External):外部高速晶振
HSI(High-Speed Internal):内部高速晶振
LSE(Low-Speed External)


MAP文件组成

| 组成部分        | 简介                           |
| ----------- | ---------------------------- |
| 程序段交叉引用关系   | 描述各文件之间函数调用关系                |
| 删除映像未使用的程序段 | 描述工程中未用到而被删除的冗余程序段(函数/数据)    |
| 映像符号表       | 描述各符号（程序段/数据）在存储器中的地址、类型、大小等 |
| 映像内存分布图     | 描述各个程序段（函数）在存储器中的地址及占用大小     |
| 映像组件大小      | 给出整个映像代码（.o）占用空间汇总信息         |
