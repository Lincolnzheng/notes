window watchdog,窗口看门狗
能产生系统复位信号和提前唤醒中断的计数器

特性：当递减计数器从0x40减到0x3F时复位（T6位跳变到0）
	   计数器的值大于`W[6:0]`值时喂狗复位
	   提前唤醒中断：递减计数器等于0x40时可产生中断
喂狗：在窗口期内喂狗。
![[Pasted image 20260812100107.png]]
WWDG初始化配置步骤
1,WWDG工作参数初始化          `HAL_WWDG_Init()`
2,WWDG Msp初始化                `HAL_WWDG_MspInit()`
3,设置优先级，使能中断            `HAL_NVIC_SetPriority()` `HAL_NVIC_EnableIRQ()`
4,编写中断处理函数                   `WWDG_IRQHandler() ->HAL_WWDG_IRQHandler`
5,重定义提前唤醒回调函数         `HAL_WWDG_EarlyWakeupCallback()`
6,在窗口期内喂狗                      `HAL_WWDG_Refresh()`


## 相关笔记
- [[IWDG]]
- [[TIMER]]