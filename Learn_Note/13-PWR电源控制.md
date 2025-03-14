# 13 PWR电源控制

## 13.1.1 PWR简介

PWR（Power Control）电源控制

> 1、PWR负责管理STM32内部的电源供电部分，可以实现可编程电压监测器和低功耗模式的功能
> 2、可编程电压监测器（PVD）可以监控VDD电源电压，当VDD下降到PVD阀值以下或上升到PVD阀值之上时，PVD会触发中断，用于执行紧急关闭任务
>
> > 在电压过低的情况下可能会导致外部或内部不确定的错误，所以可以使用PVD来紧急关闭某些设备防止意外。
>
> 3、低功耗模式包括睡眠模式（Sleep）、停机模式（Stop）和待机模式（Standby），可在系统空闲时，降低STM32的功耗，延长设备使用时间
>
> > 对使用电池的设备，空闲状态时关闭不必要的硬件，比如直接把CPU断电或者关闭时钟让程序不再运行，保留唤醒电路，比如串口接收数据的中断唤醒、外部中断唤醒、RTC闹钟唤醒...唤醒后可以继续正常工作以达到省电的目的。所以在考虑低功耗模式时我们要考虑：关闭哪些硬件，保留哪些硬件，如何去唤醒。关闭越多的硬件，设备越省电，唤醒越麻烦（三种模式区分的地方）。

### 13.1.2 电源框图

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/13.1.2%E7%94%B5%E6%BA%90%E6%A1%86%E5%9B%BE.png?raw=true)

> 1、模拟供电：为AD转换器、温度传感器、复位模块、PLL...供电，一般引脚多的设备参考电压VREF+、VREF-是单独引出，STM32F1C8T6是直接与VDDA、VSSA连接在了一起
>
> 2、数字供电：VDD供电区域为I/O电路供电、待机电路供电，通过电压调节器降至1.8V为CPU核心存乎其、内置数字外设供电。STM32的CPU、存储器和外设都是以1.8V低电压运行的，当需要与外界交流时才通过I/O电路转换到3.3V。
>
> 3、备用供电：为LSE、后备寄存器、RCC BDCR寄存器（RCC的寄存器加偶哦备份域控制寄存器）、RTC供电。低电压检测器用来切换数字供电还是备用供电。

### 13.1.3上电复位和掉电复位

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/13.1.3%E4%B8%8A%E7%94%B5%E5%A4%8D%E4%BD%8D%E5%92%8C%E6%8E%89%E7%94%B5%E5%A4%8D%E4%BD%8D.png?raw=true)

> 当内部电路电压过低时直接产生复位，让STM32复位住不要乱操作，有一个40mV的迟滞电压，当大于上限POR（Power On Reset）时解除复位，小于下限PDR（Power Down Reset）时复位，这是一个典型的迟滞比较器上限和下限的范围大约在1.8~2.0V（请参考数据手册）

### 13.1.4 可编程电压检测器

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/13.1.4%E5%8F%AF%E7%BC%96%E7%A8%8B%E7%94%B5%E5%8E%8B%E6%A3%80%E6%B5%8B%E5%99%A8.png?raw=true)

> 监测VDD和VDDA的电压，阈值电压可由程序指定，调节范围参考数据手册2.2~2.9V左右。STM32正常工作时在3.3V当电压在2.2~2.9V可以由PVD来进行警告，当低到1.8~2.0V就到了复位电路的监测范围了。PVD输出：电压过低为1，正常为0（可以申请中断，通过外部中断实现的）。

### 13.1.5 低功耗模式

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/13.1.5%E4%BD%8E%E5%8A%9F%E8%80%97%E6%A8%A1%E5%BC%8F.png?raw=true)

> 从上到下关闭的硬件越来越多，也越来越难唤醒。
>
> 1、睡眠：
>
> - 进入与唤醒：调用WFI（Wait For Interrrupt）函数等待中断，只要有任***意中断即可唤醒***，一般醒来第一件事是处理中断函数。调用WFE（Wait For Event）函数等待事件，醒来后一般不进入中断函数，直接从睡的地方继续运行。
>
> - 对电路的影响：关闭CPU时钟，其它无影响。
>
> 由于只是关闭了时钟，1.8V区域其他的供电还在，所以数据都还在。
>
> 2、停机：
>
> - 进入与唤醒：SLEEPDEEP位置1表示要进入停机或待机模式，PDDS位用来选择是停机还是待机（0停机，1待机），LPDS位用来决定电压调节器是开启还是进入低功耗模式（0开启，1低功耗），最后调用WFI或WFE函数就可以进入停机模式。***必须使用外部中断才能唤醒***。
> - 对电路的影响：关闭所有1.8V区域的时钟，因为电源没有关闭所以寄存器数据都还在。HSI和HSE的振荡器关闭，***LSI、LSE如果开启过还会继续运行***。电压调节器开启和低功耗没有什么区别，只有在唤醒时低功耗模式唤醒需要更长时间。
>
> 3、待机
>
> - 进入与唤醒：跟停机一样，但不需要LPDS位的设置，电压调节器默认关闭。只有WKUP引脚上升沿、RTC闹钟、NRST引脚外部复位、IWDG复位才能唤醒。
> - 对电路的影响：关闭1.8V区域的时钟和电源，HIS和HSE关闭，寄存器数据丢失。***LSI、LSE如果开启过还会继续运行。***
>
> 

## 13.2.1 模式选择

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/13.2.1%E6%A8%A1%E5%BC%8F%E9%80%89%E6%8B%A9.png?raw=true)

### 13.2.2 睡眠模式

•执行完WFI/WFE指令后，STM32进入睡眠模式，程序暂停运行，唤醒后程序从暂停的地方继续运行

•SLEEPONEXIT位决定STM32执行完WFI或WFE后，是立刻进入睡眠，还是等STM32从最低优先级的中断处理程序中退出时进入睡眠

•在睡眠模式下，所有的I/O引脚都保持它们在运行模式时的状态

•WFI指令进入睡眠模式，可被任意一个NVIC响应的中断唤醒

•WFE指令进入睡眠模式，可被唤醒事件唤醒

> - 在外设控制寄存器中使能一个中断，而不是在NVIC(嵌套向量中断控制器)中使能，并且在 Cortex-M3系统控制寄存器中使能SEVONPEND位。当MCU从WFE中唤醒后，外设的中断 挂起位和外设的NVIC中断通道挂起位(在NVIC中断清除挂起寄存器中)必须被清除。
>
> - 配置一个外部或内部的EXIT线为事件模式。当MCU从WFE中唤醒后，因为与事件线对应的 挂起位未被设置，不必清除外设的中断挂起位或外设的NVIC中断通道挂起位。

### 13.2.3 停止模式

•执行完WFI/WFE指令后，STM32进入停止模式，程序暂停运行，唤醒后程序从暂停的地方继续运行

•1.8V供电区域的所有时钟都被停止，PLL、HSI和HSE被禁止，SRAM和寄存器内容被保留下来

•在停止模式下，所有的I/O引脚都保持它们在运行模式时的状态

•当一个中断或唤醒事件导致退出停止模式时，HSI被选为系统时钟

> 因此需要在唤醒后，调用函数开启HSE时钟的函数恢复72MHz的主频，否则将会以HSI的8MHz主频运行。

•当电压调节器处于低功耗模式下，系统从停止模式退出时，会有一段额外的启动延时

•WFI指令进入停止模式，可被任意一个EXTI中断唤醒

•WFE指令进入停止模式，可被任意一个EXTI事件唤醒

### 13.2.4 待机模式

•执行完WFI/WFE指令后，STM32进入待机模式，唤醒后程序从头开始运行

> 因为电源被关闭了

•整个1.8V供电区域被断电，PLL、HSI和HSE也被断电，SRAM和寄存器内容丢失，只有备份的寄存器和待机电路维持供电

•在待机模式下，所有的I/O引脚变为高阻态（浮空输入）

•WKUP引脚的上升沿、RTC闹钟事件的上升沿、NRST引脚上外部复位、IWDG复位退出待机模式

## 13.3.1 修改主频

1、库函数执行时钟初始化的流程：启动HSI->回复各种缺省配置->调用SetSysClock（分配函数，根据解除注释的宏定义来选择对应的时钟初始化函数）->执行时钟初始化函数

2、在system_stm32f10x.c文件中有两个外部可调用函数和一个外部可调用变量。

> - 函数：
>
>   SystemInit()：在main函数执行前会自动调用。
>   SystemCoreClockUpdate()：更新SystemCoreClock变量的值。
>
> - 变量：
>
> - SystemCoreClock：用来显示现在的主频，只有最开始的一次会赋值,不会随着主频更改而更改，需要调用SystemCoreClockUpdate()来更新这个值。

***由于本实验只是修改了宏定义所以不进行代码展示。***

## 13.4.1 睡眠模式+串口发送+接收

串口收发的函数请参考前面的笔记，这里只展示main.c。

```c
#include "stm32f10x.h"             
#include "Delay.h"
#include "OLED.h"
#include "Serial.h"

uint8_t RxData;

int main(void)
{
	OLED_Init();
	OLED_ShowString(1, 1, "RxData:");
	
	Serial_Init();
	
	while (1)
	{
		if (Serial_GetRxFlag() == 1)
		{
			RxData = Serial_GetRxData();
			Serial_SendByte(RxData);
			OLED_ShowHexNum(1, 8, RxData, 2);
		}
	
        //展示是运行状态还是睡眠状态
		OLED_ShowString(1,1,"Running");
		Delay_ms(500);
		OLED_ShowString(1,1,"       ");
		Delay_ms(500);

        //开启睡眠模式等待中断唤醒，这里的函数调用是固定格式
		__WFI();
        //__WFE();
	}
}

```

## 13.5.1停止模式+红外对射

红外对射的代码请请参考前面的笔记，这里仅展示main.c

```c
#include "stm32f10x.h"    
#include "Delay.h"
#include "OLED.h"
#include "CountSensor.h"

int main(void)
{
	OLED_Init();
	CountSensor_Init();
	
    //开启PWR的时钟
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_PWR,ENABLE);

	OLED_ShowString(1, 1, "Count:");
	
	while (1)
	{
		OLED_ShowNum(1, 7, CountSensor_Get(), 5);

		//展示是运行状态还是停机状态
		OLED_ShowString(1,1,"Running");
		Delay_ms(500);
		OLED_ShowString(1,1,"       ");
		Delay_ms(500);

		//进入停机模式
		PWR_EnterSTOPMode(PWR_Regulator_ON,PWR_STOPEntry_WFI);
		//使用HSE为晶振倍频至72MHz
		SystemInit();
	}
}

```

这里调用SystemInit();的原因

> 当一个中断或唤醒事件导致退出停止模式时，HSI被选为系统时钟

## 13.5.2待机模式+实时时钟

实时时钟的代码请请参考前面的笔记，这里仅展示main.c

```c
#include "stm32f10x.h"   
#include "Delay.h"
#include "OLED.h"
#include "MyRTC.h"

uint32_t Alarm_CNT;

int main(void)
{
	OLED_Init();
	MyRTC_Init();
	
	//开启PWR的时钟
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_PWR,ENABLE);

	//开启WKUP引脚唤醒
	PWR_WakeUpPinCmd(ENABLE);

	OLED_ShowString(1, 1, "CNT:");
	OLED_ShowString(2, 1, "ALR:");
	OLED_ShowString(3, 1, "ALRF:");

	//设置闹钟值、显示闹钟设定值
	Alarm_CNT=RTC_GetCounter()+10;
	RTC_SetAlarm(Alarm_CNT);
	OLED_ShowNum(2, 5,Alarm_CNT,10);

	while (1)
	{
		//显示当前计数值、闹钟标志位
		OLED_ShowNum(1, 5, RTC_GetCounter(), 10);
		OLED_ShowNum(3,6,RTC_GetFlagStatus(RTC_FLAG_ALR),1);
		RTC_ClearFlag(RTC_FLAG_ALR);

		//展示是运行状态还是睡眠状态
		OLED_ShowString(4,1,"Running");
		Delay_ms(500);
		OLED_ShowString(4,1,"       ");
		Delay_ms(500);

		//关闭外设减少耗电
		OLED_Clear();

		PWR_EnterSTANDBYMode();
	}
}

```

