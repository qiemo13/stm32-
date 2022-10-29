# STM32的学习记录
## 一.下载软件以及调试
以[野火STM32MINI标准库开发指南](https://doc.embedfire.com/mcu/stm32/f103mini/std/zh/latest/index.html)为学习基本资料，成功下载了Keil_5软件以及破解。

同时，参考上述指南，成功新建空白工程并且烧录。了解了工程的结构，函数的存放，魔术棒选项配置。

## 二.学习寄存器
在视频参考下，学习了寄存器的基本概念。我的理解是寄存器是依据功能的不同而给每个内存单元起的别名，方便人来记忆与使用。对寄存器的操作是用寄存器的地址找到对应的外设，通过这种方法对外设进行一些操作达到自己的目的。

今天的学习在参考野火的视频资料的同时，也自己上手写了一些控制寄存器的代码，加深了对于寄存器代码的理解。

### 1.工程
至少需要包括main.c，启动文件，stm32f10x.h文件，其中main.c是主函数，启动文件与stm32f10x.h文件是厂家给的，stm32f10x.h文件作用为用汇编语言定义寄存器等等基础操作。

### 2.练习：实现流水灯代码main.c

```C
#include "stm32f10x.h"

//  &= ~ 用来清零
//  |=  用来写1

void soft_delay( unsigned int count )
{
    /*软件延迟，无法准确延迟时间，但是也是与count值正相关*/
	for(; count!=0; count--);
}

int main(void)
{

	/* 配置RCC寄存器，使能GPIO口的时钟*/
	*(unsigned int *)0X40021018 |= (1<<4);

	
	/* 配置CRL寄存器，配置为推挽输出 */
	*(unsigned int *)0X40011000 |= ( 1<<(4*2) );
	
	/* 配置ODR寄存器 */
	*(unsigned int *)0X4001100C &= ~( 1<<2 );
    *(unsigned int *)0X4001100C &= ~( 1<<3 );
	
	while(1)
	{
		*(unsigned int *)0X4001100C &= ~( 1<<2 );//开D4
		*(unsigned int *)0X4001100C |= ( 1<<3 );//关D5
		soft_delay(0xfffff);
		
		*(unsigned int *)0X4001100C |= ( 1<<2 );//关D4
		*(unsigned int *)0X4001100C &= ~( 1<<3 );//开D5
		soft_delay(0xfffff);
	}
}

void SystemInit(void)
{
	/* 骗过编译器不报错 */
}

```

### 3.一些代码笔记：

``` C
*(unsigned int *)0X4001100C &= ~( 1<<2 );
```
这一句的 `&= ~`语句，起到的作用是将第三位寄存器的值置0而其余不变，适合对某一位的操作。

`*(unsigned int *)` 表示将后面的数字定义为指针并对指向值进行修改。

同样的，`|= `表示置1而其余位置不变。

### 4.理解
1. AHB、APB1、APB2是时钟使能的寄存器外设，所有外设的开关都要经由这三个之一打开。这三个寄存器分别负责一些外设的开关，AHB位比较少，负责一些时钟；APB2负责IO口、ADC口、TIM定时器等；APB1负责一些定时器等。
2. 开好时钟（APB2），设置好GPIO具体的输出模式（GPIOx_CRL配置0-7，GPIOx_CRH配置8-15），最后设置端口输出（GPIOC_ODR，GPIO_BSRR设置与清零），BSRR可以设置ODR的值，也可以清零。
3. LED的原理图如下，两个分别是GPIOC_2,GPIOC_3。高电平时灯关，低电平时灯开。

![LED原理图](https://img1.imgtp.com/2022/10/11/luaSQwh2.png)

4.GPIO输出模式有
- 开漏输出(GPIO_Mode_Out_OD)：正常输出低电平，若要输出电平就接上拉电阻。
- 开漏复用功能(GPIO_Mode_AF_OD) 
- 推挽输出(GPIO_Mode_Out_PP)：输出高和低电平都行。
- 推挽复用功能(GPIO_Mode_AF_PP)



## 三.库函数
学习固件库的知识。

### 固件库主要文件
1. 启动文件：startup_stm32f10x_hd.s
2. 外设相关：
   - stm32f10x.h(外设寄存器定义)
   - system_stm32f10x.h/c(系统初始化与配置系统时钟)
   - stm32f10_xxx.h/c(外设固件库头文件与库文件)
3. 内核相关：
   - core_cm3.h/c(内设寄存器)
   - misc.h/c(NVIC与SysTick相关函数)
4. 用户相关：
   - main.c(main函数)
   - stm32f10_it.h/c(中断服务函数)

### 自己建立库
在一个文件夹下添加一对 .c .h 文件，然后添加到工程中，并设置路径。在.c.h中写入自己想要的库，然后在main文件引用即可。下面是为了点亮前面提到的LED写的库函数。
```C
bsp_led.c:
#include "./led/bsp_led.h"

void LED_GPIO_Config(void)
{
	/* 定义GPIO结构体 */
	GPIO_InitTypeDef GPIO_InitTStruct;
	/* 第一步：打开外设时钟 */
	RCC_APB2PeriphClockCmd(LED1_GPIO_CLK|LED2_GPIO_CLK, ENABLE);
	
	/* 第二部：配置外设初始化结构体 */
	GPIO_InitTStruct.GPIO_Pin = LED1_GPIO_PIN;
	GPIO_InitTStruct.GPIO_Mode = GPIO_Mode_Out_PP;
	GPIO_InitTStruct.GPIO_Speed = GPIO_Speed_10MHz;
	
	/* 调用外设初始化函数，把配置好的结构体成员写到寄存器 */
	GPIO_Init(LED1_GPIO_PORT, &GPIO_InitTStruct);
	GPIO_SetBits(LED1_GPIO_PORT, LED1_GPIO_PIN);
	
	GPIO_InitTStruct.GPIO_Pin = LED2_GPIO_PIN;
	GPIO_Init(LED2_GPIO_PORT, &GPIO_InitTStruct);
	GPIO_SetBits(LED2_GPIO_PORT, LED2_GPIO_PIN);
}

```

```C
bsp_led.h
#ifndef __BSP_LED_H
#define __BSP_LED_H

#include "stm32f10x.h"

#define  LED1_GPIO_CLK         RCC_APB2Periph_GPIOC
#define  LED1_GPIO_PORT        GPIOC
#define  LED1_GPIO_PIN         GPIO_Pin_2

#define  LED2_GPIO_CLK         RCC_APB2Periph_GPIOC
#define  LED2_GPIO_PORT        GPIOC
#define  LED2_GPIO_PIN         GPIO_Pin_3


void LED_GPIO_Config(void);


#endif /* __BSP_LED_H */
```

利用宏定义，可以大大提高可移植性，如果需要用其他引脚GPIO只需要修改宏定义就可以了，不需要把函数修改。

[寄存器控制以及库函数控制流水灯的b站视频链接](https://www.bilibili.com/video/BV1rg411h7gz/?vd_source=07c1c766723efb50223ffecb9f1496d7)

## 四.时钟树

stm32内部存在着一个时钟树，它有着启动外设、设置频率等作用。时钟树又由多个时钟构成，相互影响，配置出APB1,APB2,AHB的时钟。

* HSE：高速的外部时钟。来源：无源晶振（4-16M），通常使用8M。
* HSI：高速的内部时钟。来源：芯片内部，大小为8M，当HSE故障时，系统时钟会自动切换到HSI，直到HSE启动成功。
* PLLCLK:锁相环时钟，来源：(HSI/2、HSE)经过倍频所得。
* SYSCLK：系统时钟，最高为72M。来源：HSI、HSE、PLLCLK。
* HCLK：AHB高速总线时钟，速度最高为72M。为AHB总线的外设提供时钟、为Cortex系统定时器提供时钟（SysTick）、为内核提供时钟（FCLK）。来源：系统时钟分频得到，一般设置HCLK=SYSCLK=72M。
* PCLK1：APB1低速总线时钟，最高为36M。为APB1总线的外设提供时钟。2倍频之后则为APB1总线的定时器2-7提供时钟，最大为72M。来源：HCLK分频得到，一般配置PCLK1=HCLK/2=36M。
* PCLK2：APB2高速总线时钟，最高为72M。为APB2总线的外设提供时钟。为APB2总线的定时器1和8提供时钟，最大为72M。来源：HCLK分频得到，一般配置PCLK1=HCLK=72M。

## 五.中断
stm32的中断是十分重要、功能强大的操作，每个外设都可以产生中断。在内核水平叫做系统异常，外设水平叫做外部中断。

实现中断最常用的是NVIC这个内核外设，它控制了全部的外部中断以及部分系统异常。两个重要的库文件分别是:core_cm3.h,misc.h。core_cm3.h涉及的是所有内核上的寄存器定义，与之相对的是stm32f10x.h是外设寄存器定义。NVIC就是在core_cm3.h里面定义的。

中断可以将正常的程序流打乱，插入一些程序，改变程序运行的顺序。其中涉及到的外设内核可以在SYM32F10xxx参考手册表55找到，NVIC的相关资料在STM32F10xxx Cortex-M3编程手册第四章。

中断最主要的就是中断优先级的配置，涉及的是NVIC->IPRx寄存器。NVIC->IPRx给每个位置提供了8bit，但是低4位不用。优先级设定对应的固件库函数是 static __INLINE void NVIC_SetPriority(IRQn_Type IRQn, uint32_t priority)。IRQn_Type IRQn对应的是向量表的位置编号，当IRQn小于0，则对应内核，大于0对应外设。uint32_t priority表示的是优先级。

优先级的分组的意义在于将4bit的数据表示出远超16种的优先级。优先级分组为5组，主优先级与子优先级的不同是分组的依据，最后可以得到256种嵌套，远大于NVIC可控制的86个位置。对应固件库函数：void NVIC_PriorityGroupConfig(uint32_t NVIC_PriorityGroup)。

中断编程顺序：

1. 使能中断请求
2. 配置中断优先级分组
3. 配置NVIC寄存器，初始化NVIC_InitTypeDef
4. 编写中断服务函数

### 1.使能中断请求
精确到外设水平，可以在具体的外设手册上查看

### 2.配置中断优先级分组
由void NVIC_PriorityGroupConfig(uint32_t NVIC_PriorityGroup) 完成。

### 3.配置NVIC寄存器，初始化NVIC_InitTypeDef
NVIC_InitTypeDef函数将之前配置的所有东西写到相关的寄存器内。
### 3.编写中断服务函数
在启动文件里面的向量表写了所有的中断与异常的中断服务函数，一定要对着表写对，否则会执行默认的空函数，之前的中断不执行。中断服务函数的位置可以写在it.c里面，方便管理。