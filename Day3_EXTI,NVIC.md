## NVIC
### NVIC基本结构
<img width="1284" height="833" alt="image" src="https://github.com/user-attachments/assets/f848ca59-c924-4e53-b36a-c9723fa69520" />

### NVIC优先级分组
- 抢占优先级高的可以中断嵌套，响应优先级高的可以优先排队，抢占优先级和响应优先级均相同的按中断号排队
<img width="1181" height="352" alt="image" src="https://github.com/user-attachments/assets/698e72d1-9bc3-4e88-80f5-3cc2b1f433f8" />

## EXTI（Extern Interrupt）外部中断
### EXTI基本结构
<img width="1524" height="833" alt="image" src="https://github.com/user-attachments/assets/5c6d2468-2992-4f83-b2d2-33798e7e1da4" />

- AFIO复用IO口
<img width="686" height="738" alt="image" src="https://github.com/user-attachments/assets/8ddd6e35-cc28-46af-beb0-e12267f0a419" />

### 对射式红外传感器计次
`CountSensor.c`
```c
#include "stm32f10x.h"                  // Device header

uint8_t Count;

void CountSensor_Init(void)
{
	//开启GPIO和AFIO时钟，注意EXTI不需要开启，而NVIC在内核RCC更管不到了
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE);
	//GPIO初始化，配置成上拉输入
	GPIO_InitTypeDef GPIO_InitStruct;
	GPIO_InitStruct.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStruct.GPIO_Pin = GPIO_Pin_14;
	GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOB, &GPIO_InitStruct);
	//AFIO-数据选择器初始化
	GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource14);
	//EXTI初始化
	EXTI_InitTypeDef EXTI_InitStruct;
	EXTI_InitStruct.EXTI_Line = EXTI_Line14;
	EXTI_InitStruct.EXTI_LineCmd = ENABLE;
	EXTI_InitStruct.EXTI_Mode = EXTI_Mode_Interrupt;
	EXTI_InitStruct.EXTI_Trigger = EXTI_Trigger_Rising;
	EXTI_Init(&EXTI_InitStruct);
	//NVIC初始化
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
	NVIC_InitTypeDef NVIC_InitStruct;
	NVIC_InitStruct.NVIC_IRQChannel = EXTI15_10_IRQn;
	NVIC_InitStruct.NVIC_IRQChannelCmd = ENABLE;
	NVIC_InitStruct.NVIC_IRQChannelPreemptionPriority = 1;
	NVIC_InitStruct.NVIC_IRQChannelSubPriority = 1;
	NVIC_Init(&NVIC_InitStruct);
}

uint8_t CountSensor_Count(void)
{
	return Count;
}

void EXTI15_10_IRQHandler(void) 
{
	if(EXTI_GetITStatus(EXTI_Line14) == SET){
		Count++;
		EXTI_ClearITPendingBit(EXTI_Line14);
	}
}

```


