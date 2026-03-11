<img width="1152" height="709" alt="image" src="https://github.com/user-attachments/assets/112d7834-1b5a-40b9-bcf1-64f77e3ba1cd" />
<img width="1363" height="527" alt="image" src="https://github.com/user-attachments/assets/d903c3aa-ab5e-4056-b053-e7242e4bde2e" />

```c
模式名称	    性质	     标准库 GPIO_Mode 枚举值
浮空输入	    数字输入	 GPIO_Mode_IN_FLOATING
上拉输入	    数字输入	 GPIO_Mode_IPU（Input Pull-Up）
下拉输入	    数字输入	 GPIO_Mode_IPD（Input Pull-Down）
模拟输入	    模拟输入	 GPIO_Mode_AIN（Analog Input）
开漏输出	    数字输出	 GPIO_Mode_Out_OD（Output Open Drain）
推挽输出	    数字输出 	 GPIO_Mode_Out_PP（Output Push-Pull）
复用开漏输出	数字输出	 GPIO_Mode_AF_OD（Alternate Function Open Drain）
复用推挽输出	数字输出	 GPIO_Mode_AF_PP（Alternate Function Push-Pull）
```

### 踩坑点
- 连用两个`if`而不是用`if else`
