# TIM（Timer）定时器
- 定时器可以对输入的时钟进行计数，并在计数值达到设定值时触发中断
- 16位计数器、预分频器、自动重装寄存器的时基单元，在72MHz计数时钟下可以实现最大59.65s的定时
- 不仅具备基本的定时中断功能，而且还包含内外时钟源选择、输入捕获、输出比较、编码器接口、主从触发模式等多种功能
- 根据复杂度和应用场景分为了高级定时器、通用定时器、基本定时器三种类型

## 定时器类型
<img width="1426" height="416" alt="image" src="https://github.com/user-attachments/assets/16b3e142-d2da-4076-8e68-587dac0bf12d" />

- STM32F103C8T6定时器资源：TIM1、TIM2、TIM3、TIM4

### 基本定时器
<img width="897" height="427" alt="image" src="https://github.com/user-attachments/assets/1f744ec0-4a4e-44e6-a497-9d05f9c5852c" />

### 通用定时器
<img width="897" height="709" alt="image" src="https://github.com/user-attachments/assets/984390a9-64cf-4c48-82ef-43378d1ad35f" />

### 高级定时器
<img width="836" height="709" alt="image" src="https://github.com/user-attachments/assets/cbff441b-fe15-4165-aca4-bc77ae8ef5c1" />

## 定时中断基本结构
<img width="1140" height="491" alt="image" src="https://github.com/user-attachments/assets/46450928-404f-4eae-84d1-58ca1bfe7213" />

- 因为定时器里有好多地方会申请中断，所以需要有`中断输出控制`模块

### 预分频器时序
<img width="1182" height="638" alt="image" src="https://github.com/user-attachments/assets/834f58e7-d44f-4b18-96f3-c2660cbb3949" />

- 计数器计数频率：CK_CNT = CK_PSC / (PSC + 1)

### 计数器时序
<img width="1182" height="512" alt="image" src="https://github.com/user-attachments/assets/79c81aae-c12b-4714-bf55-1937ea8ca8ec" />

- 计数器溢出频率：CK_CNT_OV = CK_CNT / (ARR + 1)= CK_PSC / (PSC + 1) / (ARR + 1)

#### 计数器无预装时序
<img width="1182" height="640" alt="image" src="https://github.com/user-attachments/assets/b13f0d96-a241-4379-9a3b-d8a2d8544d69" />

#### 计数器有预装时序
<img width="1182" height="711" alt="image" src="https://github.com/user-attachments/assets/51a3f892-8186-4fea-bdc8-eb6250e7270d" />

## RCC时钟树
<img width="651" height="709" alt="image" src="https://github.com/user-attachments/assets/253c4774-61e6-4d9d-bfad-52e25a2404f9" />

# demo
## 定时器定时中断
`Timer.c`
```c
#include "stm32f10x.h"                  // Device header

extern int16_t Num;      //`Num`定义在`main.c`里面

void Timer_Init(void)
{	//开启TIM时钟
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);
	//选择RCC内部时钟
	TIM_InternalClockConfig(TIM2);
	//配置时基单元
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStruct;
	TIM_TimeBaseInitStruct.TIM_ClockDivision = TIM_CKD_DIV1;		//时钟源到时基单元中间滤波器的一个参数，无影响
	TIM_TimeBaseInitStruct.TIM_CounterMode = TIM_CounterMode_Up;	//向上计数
	TIM_TimeBaseInitStruct.TIM_Period = 10000 - 1;					//ARR的值
	TIM_TimeBaseInitStruct.TIM_Prescaler = 7200 - 1;				//PSC的值
	TIM_TimeBaseInitStruct.TIM_RepetitionCounter = 0;				//高级计时器才有的
	TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStruct);
	
	
	//初始化时使Num从0开始
	TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
	
	
	//TIM中断输出控制
	TIM_ITConfig(TIM2, TIM_IT_Update, ENABLE);				//更新事件时中断标志位置1
	//NVIC初始化
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);			//2位抢占优先级2位响应优先级
	NVIC_InitTypeDef NVIC_InitStruct;
	NVIC_InitStruct.NVIC_IRQChannel = TIM2_IRQn;			//TIM2中断通道
	NVIC_InitStruct.NVIC_IRQChannelCmd = ENABLE;		
	NVIC_InitStruct.NVIC_IRQChannelPreemptionPriority = 1;	//抢占优先级设为1
	NVIC_InitStruct.NVIC_IRQChannelSubPriority = 1;			//响应优先级设为1
	NVIC_Init(&NVIC_InitStruct);
	//使能TIM
	TIM_Cmd(TIM2, ENABLE);
}

void TIM2_IRQHandler(void)
{
	if(TIM_GetITStatus(TIM2, TIM_IT_Update) == SET){
		Num++;
		TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
	}
}
```
- 跨.c文件的变量：在要使用的.c文件里用extern声明

## 定时器外部时钟
- 通过红外传感器手动模拟外部时钟

`Timer.c`
```c
#include "stm32f10x.h"                  // Device header

extern int16_t Num;

void Timer_Init(void)
{	//开启TIM和GPIO时钟
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM2, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	//配置GPIO外部时钟源
	//TIM2外部时钟对应的GPIO口是PA0
	GPIO_InitTypeDef GPIO_InitStructure;
 	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_0;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
 	GPIO_Init(GPIOA, &GPIO_InitStructure);
	//选择ETR外部时钟->外部时钟模式2
    //TIM_ETRClockMode2Config(TIMx, 预分频, 极性是否翻转, 采样频率的选择);
	TIM_ETRClockMode2Config(TIM2, TIM_ExtTRGPSC_OFF, TIM_ExtTRGPolarity_NonInverted, 0x00);
	//配置时基单元
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStruct;
	TIM_TimeBaseInitStruct.TIM_ClockDivision = TIM_CKD_DIV1;		//时钟源到时基单元中间滤波器的一个参数，无影响
	TIM_TimeBaseInitStruct.TIM_CounterMode = TIM_CounterMode_Up;	//向上计数
	TIM_TimeBaseInitStruct.TIM_Period = 10 - 1;						//ARR的值
	TIM_TimeBaseInitStruct.TIM_Prescaler = 1 - 1;					//PSC的值
	TIM_TimeBaseInitStruct.TIM_RepetitionCounter = 0;				//高级计时器才有的
	TIM_TimeBaseInit(TIM2, &TIM_TimeBaseInitStruct);
	
	//初始化时使Num从0开始
	TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
	
	//TIM中断输出控制
	TIM_ITConfig(TIM2, TIM_IT_Update, ENABLE);				//更新事件时中断标志位置1
	//NVIC初始化
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);			//2位抢占优先级2位响应优先级
	NVIC_InitTypeDef NVIC_InitStruct;
	NVIC_InitStruct.NVIC_IRQChannel = TIM2_IRQn;			//TIM2中断通道
	NVIC_InitStruct.NVIC_IRQChannelCmd = ENABLE;		
	NVIC_InitStruct.NVIC_IRQChannelPreemptionPriority = 1;	//抢占优先级设为1
	NVIC_InitStruct.NVIC_IRQChannelSubPriority = 1;			//响应优先级设为1
	NVIC_Init(&NVIC_InitStruct);
	//使能TIM
	TIM_Cmd(TIM2, ENABLE);
}

void TIM2_IRQHandler(void)
{
	if(TIM_GetITStatus(TIM2, TIM_IT_Update) == SET){
		Num++;
		TIM_ClearITPendingBit(TIM2, TIM_IT_Update);
	}
}
```

## 定时器输出比较
- OC（Output Compare）输出比较
- 输出比较可以通过比较CNT与CCR寄存器值的关系，来对输出电平进行置1、置0或翻转的操作，用于输出一定频率和占空比的PWM波形
- 每个高级定时器和通用定时器都拥有4个输出比较通道，高级定时器的前3个通道额外拥有死区生成和互补输出的功能
- CCR 捕获/比较寄存器Capture/Compare Register

### PWM简介
- PWM（Pulse Width Modulation）脉冲宽度调制
- 在具有惯性的系统中，可以通过对一系列脉冲的宽度进行调制，来等效地获得所需要的模拟参量，常应用于电机控速等领域
- 使用PWM波形，就可以在数字系统等效输出模拟量
- PWM参数：  
     频率 = 1 / TS,占空比 = TON / TS ,分辨率 = 占空比变化步距
![1](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/1.png)

### 输出比较通道(通用)
![2](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/2.png)

### 输出比较模式
![4](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/4.png)

### PWM基本结构
![3](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/3.png)

- 参数计算
- PWM频率：	Freq = CK_PSC / (PSC + 1) / (ARR + 1)
- PWM占空比：	Duty = CCR / (ARR + 1)
- PWM分辨率：	Reso = 1 / (ARR + 1)

## PWM驱动LED呼吸灯
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
- 给结构体成员赋初始值  
`main.c`
```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "PWM.h"

uint16_t i;

int main(void)
{
	OLED_Init();
	PWM_Init();
	
	while (1)
	{
		for(i=0;i<100;i++){
			PWM_SetCCR(i);
			Delay_ms(10);
		}
		for(i=0;i<100;i++){
			PWM_SetCCR(100-i);
			Delay_ms(10);
		}
	}
}
```

## 定时器输入捕获
- IC（Input Capture）输入捕获
- 输入捕获模式下，当通道输入引脚出现指定电平跳变时，当前CNT的值将被锁存到CCR中，可用于测量PWM波形的频率、占空比、脉冲间隔、电平持续时间等参数
- 每个高级定时器和通用定时器都拥有4个输入捕获通道
- 可配置为PWMI模式，同时测量频率和占空比
- 可配合主从触发模式，实现硬件全自动测量

### 频率测量
![5](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/5.png)

- 测频法：在闸门时间T内，对上升沿计次，得到N，则频率:f_x=N / T
- 测周法：两个上升沿内，以标准频率fc计次，得到N ，则频率:f_x=f_c / N
- 中界频率：测频法与测周法误差相等的频率点:f_m=√f_c / T

### 输入捕获通道
![6](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/6.png)

### 主从触发模式
![7](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/7.png)

### 输入捕获基本结构
![8](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/8.png)

### PWMI基本结构
![9](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/9.png)

## PWMI模式测量频率占空比
- 自己输出PWM波形，自己测量，可调频率和占空比
`IC.c`
```c
#include "stm32f10x.h"                  // Device header

void IC_Init(void)
{
	//开启TIM时钟
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_TIM3, ENABLE);
	//初始化GPIO PA6
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;					//上拉输入
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_6;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	//选择RCC内部时钟
	TIM_InternalClockConfig(TIM3);
	//配置时基单元
	TIM_TimeBaseInitTypeDef TIM_TimeBaseInitStruct;
	TIM_TimeBaseInitStruct.TIM_ClockDivision = TIM_CKD_DIV1;		//时钟源到时基单元中间滤波器的一个参数，无影响
	TIM_TimeBaseInitStruct.TIM_CounterMode = TIM_CounterMode_Up;	//向上计数
	TIM_TimeBaseInitStruct.TIM_Period = 65536 - 1;					//ARR的值
	TIM_TimeBaseInitStruct.TIM_Prescaler = 72 - 1;					//PSC的值
	TIM_TimeBaseInitStruct.TIM_RepetitionCounter = 0;				//高级计时器才有的
	TIM_TimeBaseInit(TIM3, &TIM_TimeBaseInitStruct);
	//初始化IC1通道一和通道二
	TIM_ICInitTypeDef TIM_ICInitStruct;
	TIM_ICInitStruct.TIM_Channel = TIM_Channel_1;					//选择IC1
	TIM_ICInitStruct.TIM_ICFilter = 0xF;							//数值越大滤波效果越好，受噪声影响小
	TIM_ICInitStruct.TIM_ICPolarity = TIM_ICPolarity_Rising;		//测周期：上升沿触发
	TIM_ICInitStruct.TIM_ICPrescaler = TIM_ICPSC_DIV1;				//不分频
	TIM_ICInitStruct.TIM_ICSelection = TIM_ICSelection_DirectTI;	//直连通道，不是交叉通道
	TIM_ICInit(TIM3, &TIM_ICInitStruct);
	TIM_PWMIConfig(TIM3, &TIM_ICInitStruct);						//此PWMI函数自动把结构体里的值取反
	//配置触发源选择和从模式
	TIM_SelectInputTrigger(TIM3, TIM_TS_TI1FP1);					//选择TI1FP1作为触发源
	TIM_SelectSlaveMode(TIM3, TIM_SlaveMode_Reset);					//硬件自动清零CCR
	//使能TIM
	TIM_Cmd(TIM3, ENABLE);
}

uint16_t IC_GetFreq(void)
{
	return 1000000/(TIM_GetCapture1(TIM3)+1);						//有点误差要+1
}

uint16_t IC_GetDuty(void)
{
	return (TIM_GetCapture2(TIM3)+1)*100/(TIM_GetCapture1(TIM3)+1);	//有点误差要+1
}
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

void PWM_SetPSC(uint16_t Prescaler)
{
	TIM_PrescalerConfig(TIM2, Prescaler, TIM_PSCReloadMode_Immediate);
}
```

`main.c`
```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "PWM.h"
#include "IC.h"

int main(void)
{
	OLED_Init();
	PWM_Init();
	IC_Init();
	
	OLED_ShowString(1, 1, "Freq:00000Hz");
	OLED_ShowString(2, 1, "Duty:00%");
	

	PWM_SetPSC(72 - 1);			//Freq = 72M/(PCS+1)/(ARR+1) = 72M/(PCS+1)/ 100
	PWM_SetCCR(20);				//Duty = CCR/(ARR+1) = CCR/ 100
	
	while (1)
	{
		OLED_ShowNum(1, 6, IC_GetFreq(), 5);
		OLED_ShowNum(2, 6, IC_GetDuty(), 2);
	}
}
```
