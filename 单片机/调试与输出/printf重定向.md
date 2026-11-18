这两个示例把 `printf` 的输出重定向到 [[USART]]。

```c
#include "stdio.h"

// 重定向 fputc 函数
int fputc(int ch, FILE *f)
{
    // 等待发送缓冲区为空（USART1示例）
    while ((USART1->SR & USART_SR_TXE) == 0);
    USART1->DR = (uint8_t)ch;
    return ch;
}





//HAL库实现版本
#include "stdio.h"

int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, 0xFFFF);
    return ch;
}

```
