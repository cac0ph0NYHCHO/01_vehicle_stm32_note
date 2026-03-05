# STM32 Day1：GPIO基础+车载车灯控制
## 1. 核心知识点
- STM32工程结构：Core（业务）+ Drivers（库）+ Project（Keil）；
- GPIO初始化四步：开时钟→配结构体→初始化→控电平；
- 推挽输出（Out_PP）：车载LED/蜂鸣器控制首选。

## 2. 车载关联
- GPIO是车载外设控制的基础（车灯/蜂鸣器/按键）；
- STM32F103是车载前装/后装设备的常用入门芯片。

## 3. 踩坑记录
- 外设必须先开时钟，否则配置无效；
- 输出模式选推挽，驱动能力满足车载外设需求。

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"						//导入延时函数

int main(void)
{
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	
	GPIO_InitTypeDef GPIO_InitStruct;
	GPIO_InitStruct.GPIO_Mode = GPIO_Mode_Out_PP;
	GPIO_InitStruct.GPIO_Pin = GPIO_Pin_0;
	GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
	
	GPIO_Init(GPIOA, &GPIO_InitStruct);
	
	while (1)
	{	
		GPIO_SetBits(GPIOA, GPIO_Pin_0);
		Delay_ms(1000);
		GPIO_ResetBits(GPIOA, GPIO_Pin_0);
		Delay_ms(1000);		
	}
}
```
