![[Pasted image 20260730145859.png]]

# F1系统架构
![[Pasted image 20260730150528.png]]
四个主动单元 + 四个被动单元 还有一个ICode没有被划分到里面，因为他没有通过总线矩阵
AHB：高级高性能总线
APB：高级外围总线   

系统架构简图：
![[Pasted image 20260730151255.png]]
AHB：72MHz(Max)
APB1：36MHz(Max)
APB2：72MHz(Max)
通过外部高速晶振来间接确定时钟频率

# F4系统架构
8个主控总线 + 7个被控总线![[Pasted image 20260730152759.png]]

![[Pasted image 20260730153527.png]]
总线时钟频率（F407）：
AHB1/2：168MHz(Max)
APB1：42MHz(Max)
APB2：84MHz(Max)

FSMC：静态存储控制器
