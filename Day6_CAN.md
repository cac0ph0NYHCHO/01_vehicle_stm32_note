# CAN简介
- CAN总线（Controller Area Network Bus）控制器局域网总线
- CAN总线是由BOSCH公司开发的一种简洁易用、传输速度快、易扩展、可靠性高的串行通信总线，广泛应用于汽车、嵌入式、工业控制等领域
- CAN总线特征：  
  两根通信线（CAN_H、CAN_L），线路少  
  差分信号通信，抗干扰能力强  
  异步，无需时钟线，通信速率由设备各自约定  
  半双工，可挂载多设备，多设备同时发送数据时通过仲裁判断先后顺序  
  11位/29位报文ID，用于区分消息功能，同时决定优先级  
  可配置1~8字节的有效载荷  
  可实现广播式和请求式两种传输方式  
  应答、CRC校验、位填充、位同步、错误处理等特性  

#### 主流通信协议对比
![26](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/26.png)

## CAN硬件电路
- 每个设备通过CAN收发器挂载在CAN总线网络上
- CAN控制器引出的TX和RX与CAN收发器相连，CAN收发器引出的CAN_H和CAN_L分别与总线的CAN_H和CAN_L相连
- 高速CAN使用闭环网络，CAN_H和CAN_L两端添加120Ω的终端电阻
- 低速CAN使用开环网络，CAN_H和CAN_L其中一端添加2.2kΩ的终端电阻
![27](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/27.png)

## CAN电平标准
- CAN总线采用差分信号，即两线电压差（VCAN_H-VCAN_L）传输数据位
- 高速CAN规定：  
	电压差为0V时表示逻辑1（隐性电平）  
	电压差为2V时表示逻辑0（显性电平）
- 低速CAN规定：  
	电压差为-1.5V时表示逻辑1（隐性电平）  
	电压差为3V时表示逻辑0（显性电平）
![28](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/28.png)

#### CAN收发器 – TJA1050（高速CAN）
![29](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/29.png)

#### CAN物理层特性
![30](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/30.png)

## 数据帧
![31](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/31.png)

- 显性（0）：Dominant（D）  
 隐性（1）：Recessive（R）  

- SOF（Start of Frame）：帧起始，表示后面一段波形为传输的数据位
- ID（Identify）：标识符，区分功能，同时决定优先级
- RTR（Remote Transmission Request ）：远程请求位，区分数据帧和遥控帧
- IDE（Identifier Extension）：扩展标志位，区分标准格式和扩展格式
- SRR（Substitute Remote Request）：替代RTR，协议升级时留下的无意义位
- r0/r1（Reserve）：保留位，为后续协议升级留下空间
- DLC（Data Length Code）：数据长度，指示数据段有几个字节
- Data：数据段的1~8个字节有效数据
- CRC（Cyclic Redundancy Check）：循环冗余校验，校验数据是否正确
- ACK（Acknowledgement）：应答位，判断数据有没有被接收方接收
- CRC/ACK界定符：为应答位前后发送方和接收方释放总线留下时间
- EOF（End of Frame ）：帧结束，表示数据位已经传输完毕

### 遥控帧
- 遥控帧无数据段，RTR为隐性电平1，其他部分与数据帧相同

![32](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/32.png)

### 错误帧
- 总线上所有设备都会监督总线的数据，一旦发现“位错误”或“填充错误”或“CRC错误”或“格式错误”或“应答错误” ，这些设备便会发出错误帧来破坏数据，同时终止当前的发送设备

![33](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/33.png)

### 过载帧
- 当接收方收到大量数据而无法处理时，其可以发出过载帧，延缓发送方的数据发送，以平衡总线负载，避免数据丢失

![34](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/34.png)

### 帧间隔
- 将数据帧和遥控帧与前面的帧分离开

![35](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/35.png)

### 位填充
- 位填充规则：发送方每发送5个相同电平后，自动追加一个相反电平的填充位，接收方检测到填充位时，会自动移除填充位，恢复原始数据
- 例如：  
          即将发送：	100000110	   10000011110	    0111111111110  
          实际发送：	100000**1**110	   100000**1**1111**0**0	011111**0**11111**0**10  
          实际接收：	100000**1**110	   100000**1**1111**0**0	011111**0**11111**0**10  
          移除填充后：  100000110	   10000011110	    0111111111110  
- 位填充作用：  
 增加波形的定时信息，利于接收方执行“再同步”，防止波形长时间无变化，导致接收方不能精确掌握数据采样时机  
 将正常数据流与“错误帧”和“过载帧”区分开，标志“错误帧”和“过载帧”的特异性  
 保持CAN总线在发送正常数据流时的活跃状态，防止被误认为总线空闲  

### 接收方数据采样
- CAN总线没有时钟线，总线上的所有设备通过约定波特率的方式确定每一个数据位的时长
- 发送方以约定的位时长每隔固定时间输出一个数据位
- 接收方以约定的位时长每隔固定时间采样总线的电平，输入一个数据位
- 理想状态下，接收方能依次采样到发送方发出的每个数据位，且采样点位于数据位中心附近
- `接收方数据采样遇到的问题:`  
接收方以约定的位时长进行采样，但是采样点没有对齐数据位中心附近  
接收方刚开始采样正确，但是时钟有误差，随着误差积累，采样点逐渐偏离

## 位时序
- 为了灵活调整每个采样点的位置，使采样点对齐数据位中心附近，CAN总线对每一个数据位的时长进行了更细的划分，分为同步段（SS）、传播时间段（PTS）、相位缓冲段1（PBS1）和相位缓冲段2（PBS2），每个段又由若干个最小时间单位（Tq）构成
![36](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/36.png)

### 硬同步
- 每个设备都有一个位时序计时周期，当某个设备（发送方）率先发送报文，其他所有设备（接收方）收到SOF的下降沿时，接收方会将自己的位时序计时周期拨到SS段的位置，与发送方的位时序计时周期保持同步
- 硬同步只在帧的第一个下降沿（SOF下降沿）有效
- 经过硬同步后，若发送方和接收方的时钟没有误差，则后续所有数据位的采样点必然都会对齐数据位中心附近
![37](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/37.png)

### 再同步
- 若发送方或接收方的时钟有误差，随着误差积累，数据位边沿逐渐偏离SS段，则此时接收方根据再同步补偿宽度值（SJW）通过加长PBS1段，或缩短PBS2段，以调整同步
- 再同步可以发生在第一个下降沿之后的每个数据位跳变边沿
![38](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/38.png)

#### 波特率计算
- 波特率 = 1 / 一个数据位的时长 = 1 / (TSS + TPTS + TPBS1 + TPBS2)
- 例如：  
	SS = 1Tq，PTS = 3Tq，PBS1 = 3Tq，PBS2 = 3Tq  
	Tq = 0.5us  
	波特率 = 1 / (0.5us + 1.5us + 1.5us + 1.5us) = 200kbps  

## 多设备同时发送遇到的问题
- CAN总线只有一对差分信号线，同一时间只能有一个设备操作总线发送数据，若多个设备同时有发送需求，该如何分配总线资源？
- 解决问题的思路：制定资源分配规则，依次满足多个设备的发送需求，确保同一时间只有一个设备操作总线

### 资源分配规则1 - 先占先得
- 若当前已经有设备正在操作总线发送数据帧/遥控帧，则其他任何设备不能再同时发送数据帧/遥控帧（可以发送错误帧/过载帧破坏当前数据）
- 任何设备检测到连续11个隐性电平，即认为总线空闲，只有在总线空闲时，设备才能发送数据帧/遥控帧
- 一旦有设备正在发送数据帧/遥控帧，总线就会变为活跃状态，必然不会出现连续11个隐性电平，其他设备自然也不会破坏当前发送
- 若总线活跃状态其他设备有发送需求，则需要等待总线变为空闲，才能执行发送需求

### 资源分配规则2 - 非破坏性仲裁
- 若多个设备的发送需求同时到来或因等待而同时到来，则CAN总线协议会根据ID号（仲裁段）进行非破坏性仲裁，ID号小的（优先级高）取到总线控制权，ID号大的（优先级低）仲裁失利后将转入接收状态，等待下一次总线空闲时再尝试发送
- 实现非破坏性仲裁需要两个要求：  
线与特性：总线上任何一个设备发送显性电平0时，总线就会呈现显性电平0状态，只有当所有设备都发送隐性电平1时，总线才呈现隐性电平1状态，即：0 & X & X = 0，1 & 1 & 1 = 1  
回读机制：每个设备发出一个数据位后，都会读回总线当前的电平状态，以确认自己发出的电平是否被真实地发送出去了，根据线与特性，发出0读回必然是0，发出1读回不一定是1
- 填充位不会影响仲裁

#### 非破坏性仲裁过程
- 数据位从前到后依次比较，出现差异且数据位为1的设备仲裁失利
![39](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/39.png)

#### 数据帧和遥控帧的优先级
- 数据帧和遥控帧ID号一样时，数据帧的优先级高于遥控帧
![40](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/40.png)

#### 标准格式和扩展格式的优先级
- 标准格式11位ID号和扩展格式29位ID号的高11位一样时，标准格式的优先级高于扩展格式（SRR必须始终为1，以保证此要求）
![41](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/41.png)

- 标准遥控帧(win) VS 扩展数据帧


### 错误类型
- 错误共有5种： 位错误、填充错误、CRC错误、格式错误、应答错误
![42](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/42.png)

### 错误状态
- 主动错误状态的设备正常参与通信并在检测到错误时发出主动错误帧
- 被动错误状态的设备正常参与通信但检测到错误时只能发出被动错误帧
- 总线关闭状态的设备不能参与通信
- 每个设备内部管理一个TEC和REC，根据TEC和REC的值确定自己的状态
- TEC: Transmit Error Counter 发送错误计数器  
  REC: Receive Error Counter 接收错误计数器
![43](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/43.png)

### 错误计数器
![44](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/44.png)

# STM32 CAN外设简介
- STM32内置bxCAN外设（CAN控制器），支持CAN2.0A和2.0B，可以自动发送CAN报文和按照过滤器自动接收指定CAN报文，程序只需处理报文数据而无需关注总线的电平细节
- 波特率最高可达1兆位/秒
- 3个可配置优先级的发送邮箱
- 2个3级深度的接收FIFO
- 14个过滤器组（互联型28个）
- 时间触发通信、自动离线恢复、自动唤醒、禁止自动重传、接收FIFO溢出处理方式可配置、发送优先级可配置、双CAN模式
- STM32F103C8T6 CAN资源：CAN1

#### CAN网拓扑结构
![45](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/45.png)

#### CAN收发器电路
![46](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/46.png)

#### CAN框图
![47](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/47.png)

## CAN基本结构
![48](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/48.png)

### 发送过程
- 基本流程：选择一个空置邮箱→写入报文 →请求发送
![49](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/49.png)

### 接收过程
- 基本流程：接收到一个报文→匹配过滤器后进入FIFO 0或FIFO 1→CPU读取
![50](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/50.png)

#### 发送和接收配置位
- NART：置1，关闭自动重传，CAN报文只被发送1次，不管发送的结果如何（成功、出错或仲裁丢失）；置0，自动重传，CAN硬件在发送报文失败时会一直自动重传直到发送成功
- TXFP：置1，优先级由发送请求的顺序来决定，先请求的先发送；置0，优先级由报文标识符来决定，标识符值小的先发送（标识符值相等时，邮箱号小的报文先发送）
- RFLM：置1，接收FIFO锁定，FIFO溢出时，新收到的报文会被丢弃；置0，禁用FIFO锁定，FIFO溢出时，FIFO中最后收到的报文被新报文覆盖

## 标识符过滤器
- 每个过滤器的核心由两个32位寄存器组成：R1[31:0]和R2[31:0]
- FSCx(位宽设置): 置0，16位；置1，32位
- FBMx(模式设置)： 置0，屏蔽模式；置1，列表模式
- FFAx(关联设置)：置0，FIFO 0；置1，FIFO 1
- FACTx(激活设置)：置0，禁用；置1，启用
![51](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/51.png)

### 测试模式
- 静默模式：用于分析CAN总线的活动，不会对总线造成影响
- 环回模式：用于自测试，同时发送的报文可以在CAN_TX引脚上检测到
- 环回静默模式：用于热自测试，自测的同时不会影响CAN总线
![52](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/52.png)

### 工作模式
- 初始化模式：用于配置CAN外设，禁止报文的接收和发送
- 正常模式：配置CAN外设后进入正常模式，以便正常接收和发送报文
- 睡眠模式：低功耗，CAN外设时钟停止，可使用软件唤醒或者硬件自动唤醒
- AWUM：置1，自动唤醒，一旦检测到CAN总线活动，硬件就自动清零SLEEP，唤醒CAN外设；置0，手动唤醒，软件清零SLEEP，唤醒CAN外设
![53](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/53.png)

### 位时间特性
![54](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/54.png)

### 中断
![55](https://cdn.jsdelivr.net/gh/cac0ph0NYHCHO/01_vehicle_stm32_note@main/images/55.png)

### 时间触发通信
<img width="1844" height="909" alt="image" src="https://github.com/user-attachments/assets/fc1e4a25-abbf-4eac-b679-7820ab9e59f4" />

### 错误处理和离线恢复
- TEC和REC根据错误的情况增加或减少
- ABOM：置1，开启离线自动恢复，进入离线状态后，就自动开启恢复过程；置0，关闭离线自动恢复，软件必须先请求进入然后再退出初始化模式，随后恢复过程才被开启
<img width="781" height="460" alt="image" src="https://github.com/user-attachments/assets/e1e34f17-89bf-40c0-8b7e-da4079f6bd90" />

#### CAN_InitStruct里面参数的含义
- `CAN_Mode`选择测试模式：**！！！官方文件指引有误！！！**  
 CAN_Mode_Normal 正常模式  
 CAN_Mode_LoopBack 环回模式  
 CAN_Mode_Silent 静默模式  
 CAN_Mode_Silent_LoopBack 环回静默模式
- `CAN_Prescaler`位时序配波特率里的分频系数
- `CAN_BS1`配置BS1的Tq数量
- `CAN_BS2`配置BS2的Tq数量
- `CAN_SJW`再同步补偿宽度
- `CAN_NART`不自动重传：1关闭自动重传；0自动重传
- `CAN_TXFP`发送邮箱优先级：1先请求先发送；0ID号小的先发送
- `CAN_RFLM`FIFO锁定：1FIFO溢出时新报文丢弃；0FIFO溢出时最后收到的报文被新报文覆盖
- `CAN_AWUM`自动唤醒：1自动唤醒；0手动唤醒
- `CAN_TTCM`时间触发通信模式：1开启；0关闭
- `CAN_ABOM`离线自动恢复：1开启；0关闭

#### CAN_FilterInitStruct里面参数的含义
- `CAN_FilterNumber`指定第几个过滤器被初始化
- `CAN_FilterIdHigh` `CAN_FilterIdLow` `CAN_FilterMaskIdHigh` `CAN_FilterMaskIdLow`：过滤器的两个32位寄存器    
 16位列表模式：分别存一组ID  
 16位屏蔽模式：IdHigh存第一组ID，MaskIdHigh存屏蔽位；IdLow存第二组ID，MaskIdLow存屏蔽位  
 32位列表模式：IdHigh和IdLow存第一组ID；MaskIdHigh和MaskIdLow存第二组ID  
 32位屏蔽模式：IdHigh和IdLow存第一组ID；MaskIdHigh和MaskIdLow存屏蔽位
- `CAN_FilterScale`选择过滤器位宽：16位/32位
- `CAN_FilterMode`选择过滤器模式：列表模式/屏蔽模式
- `CAN_FilterFIFOAssignment`配置过滤器关联：进FIFO0/FIFO1
- `CAN_FilterActivation`激活过滤器，使能工作

#### TxMessage
- CRC由硬件自动生成和校验

### demo
`MyCAN.c`
```c
#include "stm32f10x.h"                  // Device header

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
	//初始化CAN1外设
	CAN_InitTypeDef CAN_InitStruct;
	CAN_InitStruct.CAN_Mode = CAN_Mode_Normal; 	//官方文件指引有误！
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

//判断FIFO里是否有数据要读，返回1表示有数据，返回0表示没有数据
uint8_t MyCAN_FIFOStatus(void)
{
	if(CAN_MessagePending(CAN1, CAN_FIFO0) > 0)
	{
		return 1;
	}
	return 0;
}

//接收报文，多返回值用指针返回
void MyCAN_Receive(uint32_t *ID, uint8_t *Length, uint8_t *Data)
{
	CanRxMsg RxMessage;
	CAN_Receive(CAN1, CAN_FIFO0, &RxMessage);	//接收报文的数据都存在结构体里了
	
	if(RxMessage.IDE == CAN_Id_Standard)
	{
		*ID = RxMessage.StdId;
	}
	else
	{
		*ID = RxMessage.ExtId;
	}
	if(RxMessage.RTR == CAN_RTR_Data)
	{
		*Length = RxMessage.DLC;
		for(uint8_t i = 0; i<*Length; i++)
		{
			Data[i] = RxMessage.Data[i];
		}
	}
	else
	{
		//......
	}
}
```
