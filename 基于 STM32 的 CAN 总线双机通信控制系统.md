- 加个呼吸灯表示待机状态：PWM输出
- 加个EXTI，光电闸门，表示紧急情况
- 加第二块STM32，使用CAN通信
- 第二块STM32使用串口输出打印日志到电脑
- 第一块OLED显示内部参数，第二块用于人机交互，OLED显示状态：正常/紧急...

### 3.26加入新内容
1.TIM3提供非阻塞延时函数：`Timer.c` `Timer.h`  
2.TIM2提供PWM波形供呼吸灯使用：`PWM.c` `PWM.h`  
3.红外传感器提供外部中断：`IRSensor.c` `IRSensor.h`  
4.LED模块完成：`LED.c` `LED.h`    
正常待机状态：蓝色LED呈现呼吸灯效果，非阻塞延迟  
发送CAN报文：蓝色LED停止呼吸，红色LED200ms慢闪600ms  
外部中断（模拟紧急情况）：蓝色LED停止呼吸，红色LED100ms快闪600ms   
5.按键消抖延时也实现非阻塞

#### 源代码
`Timer.c`
```c
#include "stm32f10x.h"

static uint32_t uwTick = 0;

void Timer_Init(void)
{
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM3, ENABLE);

    TIM_TimeBaseInitTypeDef TIM_TimeBaseStructure;
    TIM_TimeBaseStructure.TIM_Period = 72-1;    // 1us
    TIM_TimeBaseStructure.TIM_Prescaler = 1000-1; // 1ms中断
    TIM_TimeBaseStructure.TIM_ClockDivision = TIM_CKD_DIV1;
    TIM_TimeBaseStructure.TIM_CounterMode = TIM_CounterMode_Up;
    TIM_TimeBaseInit(TIM3, &TIM_TimeBaseStructure);

    TIM_ITConfig(TIM3, TIM_IT_Update, ENABLE);
    TIM_Cmd(TIM3, ENABLE);

    NVIC_InitTypeDef NVIC_InitStructure;
    NVIC_InitStructure.NVIC_IRQChannel = TIM3_IRQn;
    NVIC_InitStructure.NVIC_IRQChannelPreemptionPriority = 1;
    NVIC_InitStructure.NVIC_IRQChannelSubPriority = 1;
    NVIC_InitStructure.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&NVIC_InitStructure);
}

void TIM3_IRQHandler(void)
{
    if (TIM_GetITStatus(TIM3, TIM_IT_Update) != RESET)
    {
        uwTick++;
        TIM_ClearITPendingBit(TIM3, TIM_IT_Update);
    }
}

uint32_t GetTick(void)
{
    return uwTick;
}
```

`Timer.h`
```c
#ifndef __TIMER_H
#define __TIMER_H


void Timer_Init(void);
uint32_t GetTick(void);

#endif
```

`PWM.c`
```c
#include "stm32f10x.h"                  // Device header

void PWM_Init(void)
{	//开启TIM时钟
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);
	//初始化GPIO PA0
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;					//复用推挽输出
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	//选择RCC内部时钟
	TIM_InternalClockConfig(TIM2);
	//配置时基单元
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStruct;
	TIM_TimeBaseInitStruct.TIM_ClockDivision = TIM_CKD_DIV1;		//时钟源到时基单元中间滤波器的一个参数，无影响
	TIM_TimeBaseInitStruct.TIM_CounterMode = TIM_CounterMode_Up;	//向上计数
	TIM_TimeBaseInitStruct.TIM_Period = 100 - 1;					//ARR的值
	TIM_TimeBaseInitStruct.TIM_Prescaler = 720 - 1;					//PSC的值
	TIM_TimeBaseInitStruct.TIM_RepetitionCounter = 0;				//高级计时器才有的
	TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStruct);
	//使能TIM
	TIM_Cmd(TIM2, ENABLE);
	//配置输出比较单元OC1
	TIM_OCInitTypeDef TIM_OCInitStruct;
	TIM_OCStructInit(&TIM_OCInitStruct);						//给结构体赋默认值
	TIM_OCInitStruct.TIM_OCMode = TIM_OCMode_PWM1;				//PWM1模式1
	TIM_OCInitStruct.TIM_OCPolarity = TIM_OCPolarity_High;		//极性不反转
	TIM_OCInitStruct.TIM_OutputState = TIM_OutputState_Enable;	//使能
	TIM_OCInitStruct.TIM_Pulse = 50;							//CCR的值
	TIM_OC1Init(TIM2, &TIM_OCInitStruct);
}

void PWM_SetCCR(uint16_t Compare)
{
	TIM_SetCompare1(TIM2, Compare);
}
```

`PWM.h`
```c
#ifndef __PWM_H
#define __PWM_H

void PWM_Init(void);
void PWM_SetCCR(uint16_t Compare);

#endif
```
`IRSensor.c`
```c
#include "stm32f10x.h"                  // Device header
#include "Timer.h"

// 红灯状态定义
typedef enum {
	RED_OFF,      	// 熄灭
	RED_BLINK_SLOW,  // 慢闪（发送数据）
	RED_BLINK_FAST   // 快闪（异常/紧急）
} RedLED_State;

extern RedLED_State red_state;
extern uint32_t send_keep_timer;

void IRSensor_Init(void)
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

void EXTI15_10_IRQHandler(void) 
{
	if(EXTI_GetITStatus(EXTI_Line14) == SET){
		send_keep_timer = GetTick();
		red_state = RED_BLINK_FAST;
		EXTI_ClearITPendingBit(EXTI_Line14);
	}
}
```

`IRSensor.h`
```c
#ifndef __IRSENSOR_H
#define __IRSENSOR_H

void IRSensor_Init(void);

#endif
```

`LED.c`
```c
#include "stm32f10x.h"                  // Device header
#include "LED.h"
#include "PWM.h"

static void BlueLED_Init(void)
{
	PWM_Init();
}

static void RedLED_Init(void)
{
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;					
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_1;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	GPIO_SetBits(GPIOA, GPIO_Pin_1);
}

void LED_Init(void)
{
	BlueLED_Init();
	RedLED_Init();
}

//红灯翻转
void RedLED_Toggle(void)
{
	if(GPIO_ReadOutputDataBit(GPIOA, GPIO_Pin_1) == SET)
    {
        GPIO_ResetBits(GPIOA, GPIO_Pin_1);
    }
    else
    {
        GPIO_SetBits(GPIOA, GPIO_Pin_1);   
    }
}

//红灯熄灭
void RedLED_OFF(void)
{
	GPIO_SetBits(GPIOA, GPIO_Pin_1);
}
```

`LED.h`
```c
#ifndef __LED_H
#define __LED_H


void LED_Init(void);
void RedLED_Toggle(void);
void RedLED_OFF(void);


#endif
```

`main.c`
```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "Key.h"
#include "MyCAN.h"
#include "LED.h"
#include "Timer.h"
#include "IRSensor.h"

// 红灯状态定义
typedef enum {
	RED_OFF,      	// 熄灭
	RED_BLINK_SLOW,  // 慢闪（发送数据）
	RED_BLINK_FAST   // 快闪（异常/紧急）
} RedLED_State;

uint16_t pwm_val = 0;	//当前亮度
uint8_t dir = 0;		//0=变亮 1=变暗
uint32_t last_tick = 0;	//记录时间
	

RedLED_State red_state = RED_OFF;   // 初始熄灭
uint32_t red_timer = 0;
uint32_t send_keep_timer = 0; // 发送完成后保持闪烁的时间


int main(void)
{
	OLED_Init();
	Key_Init();
	MyCAN_Init();
	LED_Init();
	Timer_Init();
	IRSensor_Init();
	
	uint32_t KeyNum;
	
	uint32_t Tx_ID = 0x555;
	uint8_t Tx_Length = 4;
	uint8_t Tx_Data[8] = {0x11, 0x22, 0x33, 0x44};
	
	uint32_t Rx_ID;
	uint8_t Rx_Length;
	uint8_t Rx_Data[8]; 
	
	OLED_ShowString(1,1,"TxID:");
	OLED_ShowHexNum(1,6,Tx_ID,3);
	OLED_ShowString(2,1,"RxID:");
	OLED_ShowString(3,1,"Leng:");
	OLED_ShowString(4,1,"Data:");
	
	while (1)
	{	
		//正常待机状态：蓝色LED呈现呼吸灯效果，非阻塞延迟
		if(red_state == RED_OFF)
		{
			if (GetTick() - last_tick >= 10)  // 每10ms执行一次
			{
				last_tick = GetTick();
				if (dir == 0)  // 变亮
				{
					pwm_val++;
					if (pwm_val >= 100)
					{
						dir = 1;
					}
				}
				else  // 变暗
				{
					pwm_val--;
					if (pwm_val <= 0)
					{
						dir = 0;
					}
				}
				PWM_SetCCR(pwm_val);  // 输出PWM
			}
		}
		
		// 红灯 非阻塞 
		if(red_state != RED_OFF)
		{
			// 慢闪 200ms
			if(red_state == RED_BLINK_SLOW && (GetTick() - red_timer >= 200))
			{
				red_timer = GetTick();
				RedLED_Toggle();
			}

			// 快闪 100ms
			if(red_state == RED_BLINK_FAST && (GetTick() - red_timer >= 100))
			{
				red_timer = GetTick();
				RedLED_Toggle();
			}
		}
		
		KeyNum = Key_GetNum();
		if(KeyNum == 1)
		{
			red_state = RED_BLINK_SLOW;   // 开始慢闪
            send_keep_timer = GetTick();  // 记录开始时间
			Tx_Data[0]++;
			Tx_Data[1]++;
			Tx_Data[2]++;
			Tx_Data[3]++;
			MyCAN_Transmit(Tx_ID, Tx_Length, Tx_Data);
		}
		
		if(RxFlagStatus == 1)
		{
			RxFlagStatus = 0;
			if(RxMessage.IDE == CAN_Id_Standard)
			{
				Rx_ID = RxMessage.StdId;
			}
			else
			{
				Rx_ID = RxMessage.ExtId;
			}
			if(RxMessage.RTR == CAN_RTR_Data)
			{
				Rx_Length = RxMessage.DLC;
				for(uint8_t i = 0; i<Rx_Length; i++)
				{
					Rx_Data[i] = RxMessage.Data[i];
				}
			}
			else
			{
				//......
			}		
			
			OLED_ShowHexNum(2,6,Rx_ID,3);
			OLED_ShowHexNum(3,6,Rx_Length,1);
			OLED_ShowHexNum(4,6,Rx_Data[0],2);
			OLED_ShowHexNum(4,9,Rx_Data[1],2);
			OLED_ShowHexNum(4,12,Rx_Data[2],2);
			OLED_ShowHexNum(4,15,Rx_Data[3],2);
		}
		
		// 慢闪 OR 快闪 → 时间到都自动关闭！
		if ((red_state == RED_BLINK_SLOW || red_state == RED_BLINK_FAST) && 
			(GetTick() - send_keep_timer >= 600))
		{
			red_state = RED_OFF;
			RedLED_OFF();
			red_timer = 0;
		}
	}
}
```
