nested vectored interrupt controller：嵌套向量中断控制器
ISER:中断使能寄存器
ICER:中断除能寄存器
AIRCR:应用程序中断及复位控制寄存器
IPR:中断优先级寄存器
中断触发步骤
(ISER ICER)----->IPR----->CPU

配置步骤
![[Pasted image 20260805110821.png]]、

AIRCR：设置抢占响应优先级的位数
IPR：设置优先级

## 相关笔记
- [[中断的使用]]
- [[EXTI]]