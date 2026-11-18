# IAP：在应用编程

## STM32 常见编程方式

- ISP（In-System Programming）：进入芯片系统 Bootloader，通过芯片支持的通信接口和官方工具烧录。
- ICP（In-Circuit Programming）：使用调试器，通过 SWD/JTAG 接口烧录。
- IAP（In-Application Programming）：由用户程序或自定义 Bootloader 接收固件并更新 Flash，常用于现场或远程升级。

## IAP 基本流程

1. Bootloader 判断是否需要升级。
2. 通过 UART、CAN、USB 等接口接收固件。
3. 校验固件并擦除、写入应用区 Flash。
4. 再次检查应用镜像有效性。
5. 关闭中断和已使用外设，设置 MSP 与向量表，跳转到[[STM32启动|应用复位入口]]。

> [!warning]
> Flash 和 SRAM 地址范围取决于具体芯片，不能把某一型号的地址直接用于其他 STM32。分区、擦除粒度、双 Bank 能力也应查数据手册和参考手册。
