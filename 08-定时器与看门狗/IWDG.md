作用是保护系统的一个外设，用LSI来做自己的时钟源，目的是在主时钟主系统崩溃之后依然有能力将系统复位。

计数器的值到0会产生复位

Independent watchdog，独立看门狗
硬件看门狗：由硬件自动复位
软件看门狗：受软件控制复位

![[Pasted image 20260812090335.png]]
IWDG配置步骤：
![[Pasted image 20260812090548.png]]
1.取消PR/RLR寄存器写保护，设置IWDG预分频系数和重装载值，启动IWDG  `HAL_IWDG_Init()`
2.及时喂狗，写入0xAAAA到IWDG_KR      `HAL_IWDG_Refresh()`

`HAL_IWDG_Init()`:使能IWDG设置预分频系数和重装载值等
`HAL_IWDG_Refresh()`：把重装载寄存器的值重载到计数器中，喂狗

