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

