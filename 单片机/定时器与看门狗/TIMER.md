[[时钟树|时钟源]]CLK------>预分频器PSC------->计数器CNT--------->溢出


- STM32定时器
  - 常规定时器
    - 基本定时器：没有输入输出通道，常用作时基，即定时功能
    - 通用定时器：具有多路独立通道，可用作输入捕获/输出比较，也可用作时基
    - 高级定时器：除具备通用计时器的所有功能之外，还具死区控制的互补信号输出、刹车输入等功能（可用作电机控制、数字电源设计等）
  - 专用定时器
    - 独立看门狗
    - 窗口看门狗
    - 实时时钟
    - 低功耗定时器
  - 内核定时器
    - SysTick定时器

**PWM波的周期时间**
$$
T_{PWM} = \frac{(PSC + 1) \times (ARR + 1)}{F_{主频}}
\quad \text{其中} \quad
\begin{cases}
PSC: \text{预分频器值} \\
ARR: \text{自动重装载值} \\
F_{主频}: \text{单片机主时钟频率}
\end{cases}
$$
- **占空比公式**：`占空比 = CCR / (ARR + 1) × 100%`
- **其中**：
    
    - **CCR（捕获/比较寄存器）**：你写入的值，决定了信号在周期内何时翻转，是**控制占空比的直接变量**。
    - **ARR（自动重装载寄存器）**：决定了PWM信号的**周期**（频率），一般固定不变。


**CCR** ： **Capture/Compare Register 捕获/比较寄存器**。
**周期：Counter Period**
**ARR** ： **Auto-Reload Register 自动重装载寄存器**

**比较寄存器的值才是控制亮灭的关键，他通过后续配置决定了占空比**
**通过`__HAL_TIM_SET_COMPARE()`来控制**

**PWM mode 1：向上计数 CNT < CRR 时有效 CNT>=CRR 时无效**
**PWM mode 2：向下计数 CNT < CRR 时无效 CNT>=CRR 时有效**

通道模式选择

|Channel4 模式|作用|CH4 引脚是否输出|常见用途|
|---|---|---|---|
|Disable|关闭通道 4，不使用该通道|否|不需要该通道时|
|Input Capture direct mode|捕获 **CH4 引脚本身**的输入边沿时刻，计数值存入 `CCR4`|否|测量外部 PWM 的周期、频率、脉宽|
|Input Capture indirect mode|捕获与 CH4 相关联的另一个通道的输入信号，具体对应关系由定时器硬件决定|否|配合另一通道测量 PWM 高/低电平时间|
|Input Capture triggered by TRC|捕获内部触发控制器（TRC）的信号，而非直接从 CH4 引脚输入|否|定时器同步、内部触发链路测量|
|Output Compare No Output|当 `CNT == CCR4` 时发生比较匹配事件，但不改变 CH4 引脚|否|定时中断、DMA、触发 ADC/其他外设|
|Output Compare CH4|当 `CNT == CCR4` 时，按所选 OC 动作控制 CH4 引脚，如置高、置低、翻转|是|定时输出脉冲、自动闪烁 LED、方波输出|
|PWM Generation No Output|定时器内部生成 PWM 比较逻辑，但不从 CH4 引脚输出|否|用 PWM 内部事件触发 DMA、ADC 或其他定时器|
|PWM Generation CH4|在 CH4 引脚输出 PWM 波形，占空比由 `CCR4` 决定|是|LED 调光、电机调速、蜂鸣器、舵机等|
# 基本定时器

	主要特性：      
		16位递增计数器（0-65535）
		16位预分频器（分频系数1-65536）
		可用于触发DAC
		在更新事件（计数器溢出）时，可产生中断/DMA请求


框图
![[Pasted image 20260814085230.png]]

计数模式
	递增计数模式   溢出条件：CNT == ARR
	递减计数模式   溢出条件：CNT == 0
	中心对齐模式   溢出条件：CNT == ARR - 1,CNT == 1

**定时器中断配置步骤：**
	配置定时器基础工作参数       `HAL_TIM_Bash_Init()`
	定时器基础MSP初始化          `HAL_TIM_Bash_MspInit()`
	使能更新中断并启动计数器   `HAL_TIM_Base_Start_IT()`
	设置优先级，使能中断          `HAL_NVIC_SetPriority()`
	编写中断服务函数                  `TIMx_IRQHandler()等->HAL_TIM_IRQHandler()`
	编写定时器更新中断回调函数  `HAL_TIM_PeriodElapsedCallback()`


Tout​=（(ARR+1)×(PSC+1))/Tclk​​

**ARR**自动重装载寄存器值（Auto-Reload Register）
**PSC**预分频器值（Prescaler）
**T_clk**定时器输入时钟频率
**T_clk_period**定时器输入时钟周期（1/T_clk）



# 通用定时器
特性
	16位递增、递减、中心对齐计数器（计数值：0~65535）
	16位预分频器（分频系数：1~65536）
	可用于触发DAC、ADC
	在更新事件、触发事件、输入捕获、输出比较时，会产生中断/DMA请求
	4个独立通道，可用于：输入捕获、输出比较、输出PWM、单脉冲模式
	使用外部信号控制定时器且可实现多个定时器互连的同步电路
	支持编码器和霍尔传感器电路等
	定时器级联：使用一个定时器作为另一个定时器的预分频器

![[Pasted image 20260814120651.png]]



![[Pasted image 20260819115414.png]]
TI1FP1
TI1FP2
TI2FP1
TI2FP2
- **T**：代表 **Timer (定时器)**
- **I**：代表 **Input (输入)**
- **x**：代表信号来源的**物理输入通道号**（如1或2），对应芯片引脚 TIMx_CH1 和 TIMx_CH2
- **FP**：代表 **Filtered Polarity (滤波后的极性信号)**，指该信号已经通过了输入滤波器和边沿检测（极性选择）环节
## 通用定时器PWM

PWM原理图(向上计数)
![[Pasted image 20260818154437.png]]
### 通用定时器PWM输出配置步骤
```c
HAL_TIM_PWM_Init()   //初始化定时器基础参数
HAL_TIM_PWM_MspInit()  //配置NVIC,CLOCK,GPIO等
HAL_TIM_PWM_ConfigChannel() //配置PWM模式、比较值、输出极性等
HAL_TIM_PWM_Start()   //使能输出比较并启动计数器
__HAL_TIM_SET_COMPARE() //修改比较值
__HAL_TIM_ENABLE_OCxPRELOAD()  //使能通道预装载
```


### 输入捕获
通过捕获高低电平的触发事件来检测脉宽
$$
T_{pulse} = \left[ ov\_cnt \times (ARR + 1) + CCR2 - CCR1 \right] \times T_{cnt}
$$
**通用定时器输入捕获实验配置步骤：**
```c
HAL_TIM_IC_Init()        //初始化定时器基础参数
HAL_TIM_IC_MspInit()     //存放NVIC、CLOCK、GPIO初始化代码
HAL_TIM_IC_ConfigChannel()         //配置通道映射、捕获边沿、分频、滤波等
HAL_NVIC_SetPriority() , HAL_NVIC_IRQEnable()
__HAL_TIM_ENABLE_IT()       //使能更新中断等
HAL_TIM_IC_Start_IT()    //使能输入捕获、捕获中断并启动计数器
TIMx_IRQHandler() -> HAL_TIM_IRQHandler()  //定时器中断处理公用函数，处理各种中断
HAL_TIM_PeriodElaspedCallback()  //定时器更新中断回调函数，由用户重定义
HAL_TIM_IC_CaptureCallback()  //定时器输入捕获回调函数，由用户重定义
```


### 脉冲计数
**配置步骤：**
```c
HAL_TIM_IC_Init()
HAL_TIM_IC_MspInit()     //配置NVIC、CLOCK、GPIO等
HAL_TIM_SlaveConfigSynchro()  //配置定时器从模式、触发选择、分频、滤波等
//以上在cubeMX里生成但是HAL_TIM_IC_Init会用其他的代替
HAL_TIM_IC_Start()     //使能输入捕获、启动计数器
__HAL_TIM_GET_COUNTER()  //获取计数器当前值
__HAL_TIM_SET_COUNTER()  //设置计数器的值
```


# 高级定时器
### 简介
比通用寄存器多了
	重复计数器
	死区时间带可编程的互补输出
	断路输入，用于将定时器的输出信号置于用户可选的安全配置中
### 框图
![[Pasted image 20260820093309.png]]
重复计数器：溢出事件后要经过重复定时器才能产生更新中断(事件)
TIMx_CHxN：互补通道
DGT寄存器：来设置死区时间
TIMx_BKIN：断路引脚(刹车引脚)

### 特性
重复计数器(Repetition Counter Register, RCR)特性：计数器每次上溢或下溢都能使重复计数器减一，减到0时，再发生一次溢出就会产生更新事件。
![[Pasted image 20260820094930.png]]


### 配置步骤
#### 高级定时器输出指定个数PWM实验配置步骤
```c
HAL_TIM_PWM_Init() //配置定时器基础工作参数
HAL_TIM_PWM_MspInit() //NVIC,CLOCK,GPIO等的初始化
HAL_TIM_PWM_ConfigChannel() //配置PWM模式，比较值，输出极性等
HAL_NVIC_SetPriority() HAL_NVIC_EnableIQR() //设置优先级，使能NVIC中断
__HAL_TIM_ENABLE_IT() //使能定时器更新中断
HAL_TIM_PWM_Start() //使能输出主输出计数器
TIMx_IRQHandler() -> HAL_TIM_IQRHandler() //中断服务函数
HAL_TIM_PeriodElapsedCallback() //更新中断回调函数
HAL_TIM_GenerateEvent() //通过软件产生事件
```

### 高级定时器输出比较模式
输出比较模式：翻转
当CNT = CCRx，OCxREF电平反转
PWM波周期或频率由ARR决定，站恐怖固定50%，相位由CRRx决定

输出比较模式实验配置步骤
```c
HAL_TIM_OC_Init() //配置定时器基础工作参数
HAL_TIM_OC_MspInit() //NVIC,CLOCK,GPIO等的初始化
HAL_TIM_OC_ConfigChannel() //配置PWM模式，比较值，输出极性等
__HAL_TIM_ENABLE_OCxPRELOAD() //使能通道预装载
HAL_TIM_OC_Start() //使能输出比较，主输出，启动计数器
__HAL_TIM_SET_COMPARE() //修改捕获/比较寄存器的值
```

### 高级定时器互补输出带死区控制
互补输出：高低电平相反
死区：在互补信号（如PWM上下管驱动信号）的电平切换过程中，为避免两个开关管同时导通导致短路，而强制插入的一段两个信号均处于无效（关断）状态的时间间隔。（元器件有延迟特性所以要加入死区防止VCC直接接地）

**H桥：**
![[Pasted image 20260820144640.png]]
**捕获/比较通道的输出部分**
![[Pasted image 20260820145458.png]]

死区时间计算：
$$
f_{DTS} = \frac{F_t}{2^{CKD[1:0]}}
$$
TIMx_CR1:
![[Pasted image 20260821084103.png]]
判断DTG[7:5],选择计算公式

TIMx_BDTR:
![[Pasted image 20260821084346.png]]

刹车(断路)功能：


高级定时器互补输出带死区控制实验配置步骤：
```c
HAL_TIM_PWM_Init() //配置定时器基础工作参数
HAL_TIM_PWM_MspInit() //NVIC,CLOCK,GPIO等的初始化
HAL_TIM_PWM_ConfigChannel() //配置PWM模式，比较值，输出极性等
HAL_TIMEx_ConfigBreakDeadTime() //配置刹车功能死区时间
HAL_TIM_PWM_Start() //使能输出，主输出，计数器
HAL_TIMEx_PWMN_Start() //使能互补输出，主输出，计数器
```
