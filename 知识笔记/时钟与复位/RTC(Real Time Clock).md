# RTC：实时时钟

## 特性
- 能提供时间
- 能在MCU掉电后运行，由Vbat供电
- 低功耗

## 方案 

### 片上 RTC 外设方案
- 提供秒，亚秒信号
- 功耗高
- 成本低

### 独立 RTC 方案（通常通过 I²C/SPI 访问）
- 提供秒信号和日历
- 功耗低
- 体积大
- 成本高
## 常用 HAL 接口
```c
HAL_RTC_Init()
HAL_RTC_MspInit()
HAL_RCC_OscConfig()
HAL_RCCEx_PeriphCLKConfig()
HAL_RTC_Set/GetTime()
HAL_RTC_Set/GetDate()
HAL_PWR_EnableBkUpAccess()
HAL_RTCEx_BKUPWrite/Read()

//需要开启的时钟
__HAL_RCC_RTC_ENABLE()
__HAL_RCC_PWR_CLK_ENABLE()  //Power Control
__HAL_RCC_BKP_CLK_ENABLE()  //Backup Registers
```

## 相关笔记

- [[时钟树]]
- [[IIC]]
- [[LOW POWER]]
