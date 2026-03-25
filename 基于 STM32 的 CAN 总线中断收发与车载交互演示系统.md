### 源代码
`main.c`
```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "Key.h"
#include "MyCAN.h"

int main(void)
{
	OLED_Init();
	Key_Init();
	MyCAN_Init();
	
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
		KeyNum = Key_GetNum();
		if(KeyNum == 1)
		{
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
	}
}
```

`MyCAN.c`
```c
#include "stm32f10x.h"                  // Device header

CanRxMsg RxMessage;
uint8_t RxFlagStatus;


void MyCAN_Init(void)
{
	//开启GPIO和CAN1时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_CAN1, ENABLE);
	//初始化GPIO,即Rx(PA11)和Tx(PA12)
	GPIO_InitTypeDef GPIO_InitStruct;
	GPIO_InitStruct.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStruct.GPIO_Pin = GPIO_Pin_11;
	GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStruct);
	GPIO_InitStruct.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStruct.GPIO_Pin = GPIO_Pin_12;
	GPIO_InitStruct.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStruct);
	//初始化中断
	CAN_ITConfig(CAN1, CAN_IT_FMP0, ENABLE);
	
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
	
	NVIC_InitTypeDef NVIC_InitStruct;
	NVIC_InitStruct.NVIC_IRQChannel = USB_LP_CAN1_RX0_IRQn;
	NVIC_InitStruct.NVIC_IRQChannelCmd = ENABLE;
	NVIC_InitStruct.NVIC_IRQChannelPreemptionPriority = 1;
	NVIC_InitStruct.NVIC_IRQChannelSubPriority = 1;
	NVIC_Init(&NVIC_InitStruct);
	//初始化CAN1外设
	CAN_InitTypeDef CAN_InitStruct;
	CAN_InitStruct.CAN_Mode = CAN_Mode_LoopBack; 	//官方文件指引有误！
	CAN_InitStruct.CAN_Prescaler = 48;				//波特率 = 36M/48/1+2+3 = 125K
	CAN_InitStruct.CAN_BS1 = CAN_BS1_2tq;
	CAN_InitStruct.CAN_BS2 = CAN_BS2_3tq;
	CAN_InitStruct.CAN_SJW = CAN_SJW_2tq;
	CAN_InitStruct.CAN_ABOM = DISABLE;
	CAN_InitStruct.CAN_AWUM = DISABLE;
	CAN_InitStruct.CAN_NART = DISABLE;
	CAN_InitStruct.CAN_RFLM = DISABLE;
	CAN_InitStruct.CAN_TTCM = DISABLE;
	CAN_InitStruct.CAN_TXFP = DISABLE;
	CAN_Init(CAN1, &CAN_InitStruct);
	//初始化CAN1过滤器
	CAN_FilterInitTypeDef CAN_FilterInitStruct;
	CAN_FilterInitStruct.CAN_FilterNumber = 0;
	CAN_FilterInitStruct.CAN_FilterIdHigh = 0x0000;
	CAN_FilterInitStruct.CAN_FilterIdLow = 0x0000;
	CAN_FilterInitStruct.CAN_FilterMaskIdHigh = 0x0000;
	CAN_FilterInitStruct.CAN_FilterMaskIdLow = 0x0000;					//接收所有报文
	CAN_FilterInitStruct.CAN_FilterScale = CAN_FilterScale_32bit;	
	CAN_FilterInitStruct.CAN_FilterMode = CAN_FilterMode_IdMask;		//32位屏蔽模式
	CAN_FilterInitStruct.CAN_FilterFIFOAssignment = CAN_Filter_FIFO0;
	CAN_FilterInitStruct.CAN_FilterActivation = ENABLE;
	CAN_FilterInit(&CAN_FilterInitStruct);
}

//发送报文函数
void MyCAN_Transmit(uint32_t ID, uint8_t Length, uint8_t *Data)
{	
	CanTxMsg TxMessage;
	TxMessage.StdId = ID;	
	TxMessage.ExtId = ID;	
	TxMessage.IDE = CAN_Id_Standard;
	TxMessage.RTR = CAN_RTR_Data;
	TxMessage.DLC = Length;
	for(uint8_t i = 0; i<Length; i++)
	{
		TxMessage.Data[i] = Data[i];
	}
	uint8_t TransmitMailbox = CAN_Transmit(CAN1, &TxMessage);
	
	uint32_t Temp = 0;
	//等待接收完成
	while (CAN_TransmitStatus(CAN1, TransmitMailbox) != CAN_TxStatus_Ok)
	{
		//超时退出，避免程序卡死
		if(Temp > 100000)
		{
			break;
		}
		Temp++;
	}
}

//中断函数
void USB_LP_CAN1_RX0_IRQHandler(void)
{
	if(CAN_GetITStatus(CAN1, CAN_IT_FMP0) == SET)
	{
		RxFlagStatus = 1;
		CAN_Receive(CAN1, CAN_FIFO0, &RxMessage); 	//Receive函数里自动清空标志位了
													//而且CAN_ClearITPendingBit里没有CAN_IT_FMP0
													//CAN_IT_FMP0在寄存器中是2位
	}
}
```

`MyCAN.h`
```c
#ifndef __MYCAN_H
#define __MYCAN_H
 
extern CanRxMsg RxMessage;
extern uint8_t RxFlagStatus;
 
void MyCAN_Init(void);
void MyCAN_Transmit(uint32_t ID, uint8_t Length, uint8_t *Data);

#endif
```

`Key.c`
```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"

void Key_Init(void)
{
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);
	
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_1;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOB, &GPIO_InitStructure);
}

uint8_t Key_GetNum(void)
{
	uint8_t KeyNum = 0;
	if (GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_1) == 0)
	{
		Delay_ms(20);
		while (GPIO_ReadInputDataBit(GPIOB, GPIO_Pin_1) == 0);
		Delay_ms(20);
		KeyNum = 1;
	}
	return KeyNum;
}
```

`Key.h`
```c
#ifndef __KEY_H
#define __KEY_H

void Key_Init(void);
uint8_t Key_GetNum(void);

#endif
```
