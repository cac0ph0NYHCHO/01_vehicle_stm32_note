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
