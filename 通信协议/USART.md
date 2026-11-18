USART：Universal synchronous asynchronous receiver transmitter，通用同步异步收发器
UART：Universal asynchronous receiver transmitter，通用异步收发器
**异步通信 (Asynchronous)**
**同步通信 (Synchronous)**

![[Pasted image 20260806110821.png]]
接收数据：外设发送数据到RX，RX发送到接收移位寄存器，然后发送到接受数据寄存器，最后放入数据寄存器
发送数据：将数据放在DR数据寄存器，然后将数据发送到发送数据寄存器TDR，再到发送移位寄存器，再到TX发送给外设

**波特率设置**
f<sub>PCLKx</sub> : APB1/2的时钟频率
f<sub>ck</sub>：串口的时钟
![[Pasted image 20260806112558.png]]
![[Pasted image 20260806112752.png]]

通过这两个公式计算出波特率

**通讯配置步骤(USART/UART异步通讯)：**
1. 配置串口工作参数           HAL_UART_Init()
2. 串口底层初始化              HAL_UART_MspInit()
3. 开启串口异步接收中断    HAL_UART_Receive_IT()    //cubeMX 不配置，需手动配置
4. 设置优先级，使能中断    HAL_NVIC_Setpriority()、HAL_NVIC_ENABLE()
5. 编写终端服务函数           USARTx_IRQHandler()、USART_IRQHandler() 
6. 串口数据发送                  USART_DR、HAL_UART_Transmit() //cubeMX 不配置，需手动配置
句柄：包含 配置 + 状态 + 缓冲区指针 + 锁 的一个struct结构体

**回调函数`HAL_UART_RxCpltCallback`，接收发送函数`HAL_UART_Receive_IT，HAL_UART_Transmit`**
**需要自己配置，还要写接收缓冲区。中断处理函数`USARTx_IRQHandler`在stm32f4xx_it.c文件中cubeMX生成的中断函数全部都在这里面，不用自己再手动写，在这里写即可。**

**中断服务函数的编写**
![[Pasted image 20260806172024.png]]

**IO引脚复用功能：**
通用：IO端口的输入或输出是由GPIO外设控制，我们称之为通用
复用：IO端口的输入或输出是由其它非GPIO外设控制，我们称之为复用
每个端口通过配置不同的AFx来指定不同的功能
复用功能有**十六路复用器**选择，可以通过AFx来选择有复用功能的IO口不同的功能。

`硬件中断`
    `↓`
`USART1_IRQHandler()  (ISR，在 stm32f1xx_it.c)`
    `↓`
`HAL_UART_IRQHandler()  (HAL库通用处理)`
    `↓`
`UART_Receive_IT()  (内部函数，负责读数据和计数)`
    `↓`
`当 RxXferCount == 0 时  (接收完指定数量的字节)`
    `↓`
`HAL_UART_RxCpltCallback(huart)  ★ 这里回调你的函数！`
`HAL_UART_RxCpltCallback(huart)` 是一个弱定义函数，中断处理函数的逻辑在这里面定义，协议在里面定义
