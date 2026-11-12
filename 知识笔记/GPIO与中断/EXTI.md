External(Extended) interrupt/event Controller，外部(扩展)中断事件控制
![[Pasted image 20260805114035.png]] EXTI工作原理
![[Pasted image 20260805141429.png]]

**EXTI和IO的映射关系：**
**F1系列：** AFIO，Alternate Function IO，即复用功能IO，主要用于重映射和外部中断映射配置
![[Pasted image 20260805142901.png]]

**AFIO_EXTICRx**
特别别注意：配置AFIO寄存器之前要使能AFIO时钟，方法如下：__HAL_RCC_AFIO_CLK_ENABLE();

**F4系列：** SYSCFG，System configuration controller，即系统配置控制器，用于外部中断映射配置等
![[Pasted image 20260805143345.png]]
开启时钟：__HAL_RCC_SYSCFG_CLK_ENABLE(); 

**EXTI和IO映射关系：**
![[Pasted image 20260805143800.png]]

## 相关笔记
- [[中断的使用]]
- [[NVIC]]
- [[GPIO]]