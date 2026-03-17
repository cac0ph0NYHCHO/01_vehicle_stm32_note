# 通信接口
- 通信的目的：将一个设备的数据传送到另一个设备，扩展硬件系统
- 通信协议：制定通信的规则，通信双方按照协议规则进行数据收发
![10](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/10.png)

## 串口通信
- 串口是一种应用十分广泛的通讯接口，串口成本低、容易使用、通信线路简单，可实现两个设备的互相通信
- 单片机的串口可以使单片机与单片机、单片机与电脑、单片机与各式各样的模块互相通信，极大地扩展了单片机的应用范围，增强了单片机系统的硬件实力

### 硬件电路
- 简单双向串口通信有两根通信线（发送端TX和接收端RX）
- TX与RX要交叉连接
- 当只需单向的数据传输时，可以只接一根通信线
- 当电平标准不一致时，需要加电平转换芯片
![11](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/11.png)

### 电平标准
- 电平标准是数据1和数据0的表达方式，是传输线缆中人为规定的电压与数据的对应关系，串口常用的电平标准有如下三种：
- TTL电平：+3.3V或+5V表示1，0V表示0
- RS232电平：-3~-15V表示1，+3~+15V表示0
- RS485电平：两线压差+2~+6V表示1，-2~-6V表示0（差分信号）

### 串口参数及时序
- 波特率：串口通信的速率
- 起始位：标志一个数据帧的开始，固定为低电平
- 数据位：数据帧的有效载荷，1为高电平，0为低电平，低位先行
- 校验位：用于数据验证，根据数据位计算得来
- 停止位：用于数据帧间隔，固定为高电平
![12](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/12.png)

## USART简介
- USART（Universal Synchronous/Asynchronous Receiver/Transmitter）通用同步/异步收发器
- USART是STM32内部集成的硬件外设，可根据数据寄存器的一个字节数据自动生成数据帧时序，从TX引脚发送出去，也可自动接收RX引脚的数据帧时序，拼接为一个字节数据，存放在数据寄存器里
- 自带波特率发生器，最高达4.5Mbits/s
- 可配置数据位长度（8/9）、停止位长度（0.5/1/1.5/2）
- 可选校验位（无校验/奇校验/偶校验）
- 支持同步模式、硬件流控制、DMA、智能卡、IrDA、LIN
- STM32F103C8T6 USART资源： USART1、 USART2、 USART3
![13](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/13.png)

### USART基本结构
![14](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/14.png)

### 数据帧
![15](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/15.png)
![16](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/16.png)

### 起始位侦测
![17](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/17.png)

### 数据采样
![18](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/18.png)

### 波特率发生器
- 发送器和接收器的波特率由波特率寄存器BRR里的DIV确定
- 计算公式：波特率 = fPCLK2/1 / (16 * DIV)
![19](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/19.png)

### 数据模式
- HEX模式/十六进制模式/二进制模式：以原始数据的形式显示
- 文本模式/字符模式：以原始数据编码后的形式显示
![20](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/20.png)
![21](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/21.png)

## demo
### 串口发送
`Serial.c`
```c
#include "stm32f10x.h"                  // Device header

void Serial_Init(void)
{
	//开启GPIO和USART时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	//初始化GPIO PA9，发数据
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	//初始化USART1
	USART_InitTypeDef USART_InitStruct;
	USART_InitStruct.USART_BaudRate = 9600;											//波特率
	USART_InitStruct.USART_HardwareFlowControl = USART_HardwareFlowControl_None;	//硬件流控制
	USART_InitStruct.USART_Mode = USART_Mode_Tx;									//模式：发送数据
	USART_InitStruct.USART_Parity = USART_Parity_No;								//无校验位
	USART_InitStruct.USART_StopBits = USART_StopBits_1;								//1位停止位
	USART_InitStruct.USART_WordLength = USART_WordLength_8b;						//8位数据位
	USART_Init(USART1, &USART_InitStruct);
	//使能USART
	USART_Cmd(USART1, ENABLE);
}

//函数：发送一个字节
//参数：要发送的字节（数字形式）
//返回值：无
void Serial_SendByte(uint16_t Byte)
{
	USART_SendData(USART1, Byte);
	while(USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);					//TXE:发送数据寄存器空 (Transmit data register empty)
																					//确保数据已经从 发送数据寄存器 转移到 发送移位寄存器
}

//函数：发送一个数组
//参数：数组名，数组成员个数
//返回值：无
void Serial_SendArray(uint16_t *Array, uint16_t Num)
{
	uint16_t i;
	for(i = 0; i < Num; i++){
		Serial_SendByte(Array[i]);
	}
}

//函数：发送一个字符串
//参数：字符串（传入的是第一个字符的首地址）
//返回值：无
void Serial_SendString(char *String)
{
	uint8_t i;
	for(i = 0; String[i] != '\0'; i++){
		Serial_SendByte(String[i]);
	}
}

//函数：输出X的Y次方
//参数：底数X，次数Y
//返回值：X的Y次方
uint16_t Pow(uint16_t X, uint16_t Y)
{
	uint16_t temp = 1;
	while(Y!=0){
		temp *= X;
		Y--;
	}
	return temp;
}

//函数：以字符的形式发送数字
//参数：要发送的数字，数值的位数
//返回值：无
void Serial_SendNumber(uint16_t Num, uint16_t Count)
{
	uint16_t i;
	for(i=0;i<Count;i++){
		Serial_SendByte(Num/Pow(10, Count-i-1)%10 + 0x30);
	}
}
```
- `fprintf重定向`
- 勾选 MicroLIB  +  #include <stdio.h>
```c
//printf重定向
int fputc(int ch, FILE *f)
{
	Serial_SendByte(ch);
	return ch;
}
```
- 换行用`\r\n`

### 串口发送+接收
`Serial.c`
```c
#include "stm32f10x.h"                  // Device header
#include <stdio.h>

uint8_t Serial_RxData;
uint8_t Serial_RxFlag;

void Serial_Init(void)
{
	//开启GPIO和USART时钟
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_USART1, ENABLE);
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);
	//初始化GPIO PA9，发送数据
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_AF_PP;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_9;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	//初始化GPIO PA10，接收数据
	GPIO_InitStructure.GPIO_Mode = GPIO_Mode_IPU;
	GPIO_InitStructure.GPIO_Pin = GPIO_Pin_10;
	GPIO_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &GPIO_InitStructure);
	//初始化USART1
	USART_InitTypeDef USART_InitStruct;
	USART_InitStruct.USART_BaudRate = 9600;											//波特率
	USART_InitStruct.USART_HardwareFlowControl = USART_HardwareFlowControl_None;	//硬件流控制
	USART_InitStruct.USART_Mode = USART_Mode_Tx | USART_Mode_Rx;					//模式：发送数据&接收数据
	USART_InitStruct.USART_Parity = USART_Parity_No;								//无校验位
	USART_InitStruct.USART_StopBits = USART_StopBits_1;								//1位停止位
	USART_InitStruct.USART_WordLength = USART_WordLength_8b;						//8位数据位
	USART_Init(USART1, &USART_InitStruct);
	//开启USART1-RX到NVIC的中断
	USART_ITConfig(USART1, USART_IT_RXNE, ENABLE);  								//RXNE:读数据寄存器非空 (Read data register not empty)
	//NVIC初始化
	NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);
	NVIC_InitTypeDef NVIC_InitStruct;
	NVIC_InitStruct.NVIC_IRQChannel = USART1_IRQn;
	NVIC_InitStruct.NVIC_IRQChannelCmd = ENABLE;
	NVIC_InitStruct.NVIC_IRQChannelPreemptionPriority = 1;
	NVIC_InitStruct.NVIC_IRQChannelSubPriority = 1;
	NVIC_Init(&NVIC_InitStruct);
	//使能USART
	USART_Cmd(USART1, ENABLE);
}

//函数：发送一个字节
//参数：要发送的字节（数字形式）
//返回值：无
void Serial_SendByte(uint16_t Byte)
{
	USART_SendData(USART1, Byte);
	while(USART_GetFlagStatus(USART1, USART_FLAG_TXE) == RESET);					//TXE:发送数据寄存器空 (Transmit data register empty)
																					//确保数据已经从 发送数据寄存器 转移到 发送移位寄存器
}

//读取后清零Serial_RxFlag的值
uint8_t Serial_GetRxFlag(void)
{
	if(Serial_RxFlag == 1){
		Serial_RxFlag = 0;
		return 1;
	}
	return 0;
}

//返回Serial_RxData的值
uint8_t Serial_GetRxData(void)
{
	return Serial_RxData;
}

//中断函数
void USART1_IRQHandler(void)
{
	if(USART_GetITStatus(USART1, USART_IT_RXNE) == SET){
		Serial_RxFlag = 1;
		Serial_RxData = USART_ReceiveData(USART1);
		USART_ClearITPendingBit(USART1, USART_IT_RXNE);
	}
}
```
`main.c`
```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"
#include "OLED.h"
#include "Serial.h"

int main(void)
{
	OLED_Init();
	Serial_Init();
	
	OLED_ShowString(1, 1, "Num:000");
	
	while (1)
	{
		if(Serial_GetRxFlag() == 1){
			Serial_SendByte(Serial_GetRxData());
			OLED_ShowHexNum(1, 5, Serial_GetRxData(), 3);
		}
	}
}
```
