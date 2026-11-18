# ADC：模数转换器

ADC（Analog-to-Digital Converter，模数转换器）用于把模拟电压转换为数字量。

分辨率：以 STM32F4 常见的 12 位 ADC 为例，输出范围为 `0 ~ 4095`；实际能力以具体型号为准。
逐次逼近法：二分比较确定电压值，通过每位进行二分比较确定每一位是0还是1，最终确定电压值

外部通道：[[GPIO]] 口
内部通道：
	内部温度传感器
	内部参考电压


## 转换组
ADC1
	注入组
	规则组
ADC2
	注入组
	规则组
注入组最多4个
规则组最多16个

## 框图

使用 `HAL_ADC_Start_DMA()` 时，由 [[DMA]] 把转换结果写入缓冲区。
①参考电压/模拟部分电压
②输入通道
③转换序列
④触发源
⑤转换时间
⑥数据寄存器
⑦中断
![[Pasted image 20260831151628.png]]

```c
HAL_ADC_Start()
HAL_ADC_GetValue() //获取检测结果 EOC位
HAL_ADC_PollForConversion() //等待转换完成，检测EOC位(End of Conversion)
// 部分 STM32 系列提供校准接口，函数名和支持情况以该系列 HAL 为准

//ADC DMA
HAL_RCCEx_PeriphCLKConfig()
HAL_ADC_ConfigChannel()
HAL_ADC_Start_DMA()


uint32_t adc_get_result(void)
{
	HAL_ADC_Start(&hadc1);
	HAL_ADC_PollForConversion(&hadc1,10); //等待转换完成，检测EOC位(End of Conversion)
	return (uint16_t)HAL_ADC_GetValue(&hadc1); //获取检测结果
}
```
持续转换模式能让 ADC 自动开始下一次转换，但每次读取结果前仍要确认该次转换已经完成。`HAL_ADC_PollForConversion()` 是等待转换完成，不是启动连续转换。

## 触发源
ADON位触发转换(仅限F1系列)：当ADC_CR2寄存器的ADON位为1时，再单独给ADON位写1
外部事件触发转换：规则组外部触发，注入组外部触发

ADC 转换时间（F4/F7 的 12 位模式）：

`Tconv = 采样时间 + 12 个 ADC 时钟周期`

采样时间可以通过SMPx[2:0]位设置

扫描模式用于按 Rank 顺序转换多个通道。
