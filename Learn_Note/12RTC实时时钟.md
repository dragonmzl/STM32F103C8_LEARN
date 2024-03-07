# 12 RTC实时时钟

## 12.1.1 Unix时间戳

最早出现在Unix系统的时间戳，后来由Unix系统演变而来的系统也都继承了Unix时间戳的规定，目前Linux、Windows、安卓等等都是使用的Unix时间戳。

Unix 时间戳（Unix Timestamp）

> ***定义：***从UTC/GMT的1970年1月1日0时0分0秒开始所经过的秒数，不考虑闰秒（时间戳就是一个计数器数值）
>
> >这个时间戳计数到现在已经非常大了，虽然对于人类来说不易理解，但是对于计算机来说无论是存储还是计算都是非常方便的。在底层我们用时间戳来秒计数，需要给人类观看时就转换为年月日。
> >
> >***好处：***
> >
> >1、简化了硬件电路，直接设计一个很大的秒寄存器就行了
> >
> >2、在进行时间间隔的计算时很方便，只需要将两个时刻的秒数相减除以一个小时的秒数就能算出两个时刻之间的间隔。
> >
> >3、存储方便，存储秒数只需要一个比较大的变量。
> >
> >***坏处：***
> >
> >1、占用软件资源，在每次进行秒计数器和日期时间转换时会比较占用软件资源。
>
> •时间戳存储在一个秒计数器中，秒计数器为32位（2038.1.19日溢出）/64位的整型变量（STM32F103C8T6是32位的数据类型***（无符号所以会在2106年溢出）***）
>
> •世界上所有时区的秒计数器相同，不同时区通过添加偏移来得到当地时间

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/12.1.1%20Unix%E6%97%B6%E9%97%B4%E6%88%B3.png?raw=true)

### 12.1.2 UTC/GMT

GMT（Greenwich Mean Time）格林尼治标准时间是一种以地球自转为基础的时间计量系统。它将地球自转一周的时间间隔等分为24小时，以此确定计时标准/

> 但是地球自转时间会受到外界、自身影响而改变，那么这个一秒的定义也就相应变化，不利于科学研究。

UTC（Universal Time Coordinated）协调世界时是一种以原子钟为基础的时间计量系统。它规定铯133原子基态的两个超精细能级间在零磁场下跃迁辐射9,192,631,770周所持续的时间为1秒。***当原子钟计时一天的时间与地球自转一周的时间相差超过0.9秒时，UTC会执行闰秒来保证其计时与地球自转的协调一致。***

> UTC定义的一秒与1970年的GMT保持一致，但是地球的自转时间会改变，那么计时的一天就会和自转的一天发生偏差，因此引入闰秒来消除误差。

### 12.1.3 时间戳转换

C语言的time.h模块提供了时间获取和时间戳转换的相关函数，可以方便地进行秒计数器、日期时间和字符串之间的转换

<img src="https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/12.1.3%E5%87%BD%E6%95%B0.png?raw=true" style="zoom: 67%;" />

<img src="https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/12.1.3%20%E6%95%B0%E6%8D%AE%E8%BD%AC%E6%8D%A2%E7%9A%84%E5%87%BD%E6%95%B0%E8%B0%83%E7%94%A8.png?raw=true" style="zoom: 67%;" />

```c
#include <stdio.h>
#include <time.h>

time_t time_count;
struct tm time_data;
char* tm_str;

int main(void)
{
    //time_count=time(NULL);
    //time(&time_count);
    time_count=1672588795;
    printf("%d\n",time_count);

    time_data=*localtime(&time_count);
    printf("%d\n",time_data.tm_year+1900);
    printf("%d\n",time_data.tm_mon+1);
    printf("%d\n",time_data.tm_mday);
    printf("%d\n",time_data.tm_hour);
    printf("%d\n",time_data.tm_min);
    printf("%d\n",time_data.tm_sec);

    time_count=mktime(&time_data);
    printf("%d\n",time_count);

    tm_str=ctime(&time_count);
    printf(tm_str);


    tm_str=asctime(&time_data);
    printf(tm_str);

    char arr[50];
    strftime(arr,50,"%H-%M-%S",&time_data);
    printf(arr);
    return 0;
}
```

> time_t time(time_t*);
>
> > 这个函数要么使用变量放在等号左边接受，要么传入地址直接赋值。
>
>  **struct tm\*** gmtime(const **time_t*****);**  
>
>  struct tm* localtime(const  time_t*);  
>
> > 这两个函数在接收时可以使用定义***结构体指针变量***来接受，也可以定义***结构体变量***然后解引用函数的返回值来接受。

## 12.2.1 BKB备份寄存器

  BKP（Backup Registers）备份寄存器

> BKP可用于存储用户应用程序数据。当VDD（2.0~3.6V）电源被切断，他们仍然由VBAT（1.8~3.6V）维持供电。当系统在待机模式下被唤醒，或系统复位或电源复位时，他们也不会被复位
>
> > 1、当主电源掉电后，VBAT必须连接备用电池，电池的正极接VDD，电池的负极和主电源共地。
> >
> > 2、设计电路板时，如果VBAT不使用，手册建议VBAT和主电源接到一起，并且再连接一个100nF的滤波电容。
>
> TAMPER引脚产生的侵入事件将所有备份寄存器内容清除
>
> > 入侵检测提供安全保障，可以将外部一些防拆的触点接在此端口，若触点电平变化，STM32芯片会自动清空寄存器数据
>
> RTC引脚输出RTC校准时钟、RTC闹钟脉冲或者秒脉冲
>
> > RTC校准时钟：对内部RTC微小的误差进行校准
> >
> > RTC闹钟脉冲或者秒脉冲：输出出来为别的设备提供信号
>
> 存储RTC时钟校准寄存器
>
> > 用于配合RTC校准时钟进行校准
>
> 用户数据存储容量：20字节（中容量和小容量）/ 84字节（大容量和互联型）

### 12.2.2 BKP基本结构

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/12.2.2%20BKP%E5%9F%BA%E6%9C%AC%E7%BB%93%E6%9E%84.png?raw=true)

> 后备区域：BKP就在后备区域，但还有RTC和时钟配置相关电路，该区域特性时当VDD主电源掉电时，可由VBAT的备用电池供电，当VDD主电源上电时，后备区域供电切换回VDD，这样可以节省电池的电量。
>
> > 数据寄存器：用来存储数据（16位），小容量和中容量只有DR1~DR10，大容量和互联型有DR1~DR42
> >
> > RTC时钟校准寄存器：在RTC输出校准时钟时配合这个寄存器，对RTC的误差进行校准
>
> VBAT：电池供电
>
> TAMPER：侵入检测
>
> RTC：输出RTC校准时钟、RTC闹钟脉冲或者秒脉冲

## 12.3.1 RTC介绍

RTC（Real Time Clock）实时时钟

> RTC是一个独立的定时器，可为系统提供时钟和日历的功能
> RTC和时钟配置系统处于后备区域，系统复位时数据不清零，VDD（2.0~3.6V）断电后可借助VBAT（1.8~3.6V）供电继续走时
> 32位（无符号）的可编程计数器，可对应Unix时间戳的秒计数器
>
> > 读取时间时先得到秒数再用time.h的函数来转换成时间。
>
> 20位的可编程预分频器，可适配不同频率的输入时钟
>
> > 可以进行1~2^20分频输出1hz的时钟给RTCCLK
>
> 可选择三种RTC时钟源：
>  HSE时钟除以128（通常为8MHz/128）
>
> > 128分频是因为HSE的频率太快就算后面又20位分频器也很难分到1Hz
>
>  LSE振荡器时钟（通常为32.768KHz）
>
> > 32768时2^15，可以通过一个15位计数器自然溢出，溢出信号就是1Hz，不用设置计数目标，也不用比较是否记到目标值，简化了电路设计。
>
>   LSI振荡器时钟（40KHz）
>
> ***更多时候我们都使用LSE振荡器时钟，因为三个时钟源只有它可以由VBAT供电，另外两个在主电源VDD掉电后就不能工作了。***

### 12.3.2 RTC框图

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/12.3.2%20RTC%E6%A1%86%E5%9B%BE.png?raw=true)

> RTCCLK：时钟源
>
> RTC_PRL、RTC_DIV：共同构成了预分频器，通过对RTC_PRL设置目标值来确定分频系数（分频值等于写入值+1），RTC_DIV相当于一个计数器每来一个脉冲就自减一次，自减到0再来一个脉冲就会产生一个脉冲信号TR_CLK此时RTC_PRL会为RTC_DIV自动重装设置好的分频值。
>
> RTC_CNT:相当于Unix时间戳的秒计数器。
>
> RTC_ALR:设置闹钟当RTC_CNT和RTC_ALR的值一样时，产生RTC_Alarm闹钟信号，通往右边的中断系统。同时闹钟信号也可以让STM32退出待机模式。
>
> RTC_CR：中断部分，有三个信号分别是RTC_Second（来自TR_CLK每秒产生一次中断）、RTC_Overflow（32位可编程计数器RTC_CNT溢出时会产生一次中断）、RTC_Alarm。SECF、OWF、ALRF都是中断标志位，SECIE、OWIE、ALRIE都是中断使能，三个信号通过一个或门汇聚到NVIC中断控制器。
>
> WAUP：Weak Up 引脚，用来唤醒设备，下一小节学习

### 12.3.3 RTC基本结构

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/12.3.1%20RTC%E5%9F%BA%E6%9C%AC%E7%BB%93%E6%9E%84.png?raw=true)

> RTCCLK：时钟来源，需要在RCC里配置，三个时钟选择一个当作RTCCLK。
>
> 预分频器：DIV余数寄存器是一个自减寄存器，存储当前的计数值
>
> PRL重装寄存器时设定的计数目标，决定分频值，二者分频出1HZ的信号。
>
> CNT：每来一个脉冲自增一次。
>
> ALR：设置闹钟值，当与CNT一样时产生Alarm信号通向中断。

配置时钟的函数都在RCC库函数中，由于结构简单库函数配置RTC不是使用结构体配置。

### 12.3.4 RTC硬件电路

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/12.3.3%20RTC%E7%A1%AC%E4%BB%B6%E7%94%B5%E8%B7%AF.png?raw=true)

> 为了配合STM32的RTC，外部需要一些电路，在最小系统电路上，外部需要加两部分：
>
> 1、备用电池供电
>
> 2、外部低速晶振

### 12.3.5 RTC操作注意事项

1、执行以下操作将使能对BKP和RTC的访问：

 设置RCC_APB1ENR的PWREN和BKPEN，使能PWR和BKP时钟

 设置PWR_CR的DBP，使能对BKP和RTC的访问

> 第一步：开启PWR和BKP的时钟
>
> 第二步：使用PWR，使能BKP和RTC的访问

2、若在读取RTC寄存器时，RTC的APB1接口曾经处于禁止状态，则软件首先必须等待RTC_CRL寄存器中的RSF位（寄存器同步标志）被硬件置1

> RTC框图显示，PCLK1驱动APB1接口，RTCCLK驱动RTC同步寄存器，这是因为PCLK1主电源掉电后会停止工作为了RTC掉电正常工作，RTC的寄存器都是由RTCCLK同步变更的，这就会导致时钟不同步，PCLK1的频率远大于RTCCLK，如果在APB1刚开启时立刻读取RTC寄存器，有可能RTC寄存器还没有更新到APB1总线上，这样就会读数错误（通常为0）

3、必须设置RTC_CRL寄存器中的CNF位，使RTC进入配置模式后，才能写入RTC_PRL、RTC_CNT、RTC_ALR寄存器

> 库函数中已经帮我们加上了设置CNF位的代码，因此我们不需要单独调用代码进入配置模式。

4、对RTC任何寄存器的写操作，都必须在前一次写操作结束后进行。可以通过查询RTC_CR寄存器中的RTOFF状态位，判断RTC寄存器是否处于更新中。仅当RTOFF状态位是1时，才可以写入RTC寄存器

> 根本原因还是PCLK，RTCCLK的时钟频率不一样，用PCLK的频率写入之后，这个值不能立刻更新到RTC寄存器中，因此需要等待一下。

## 12.4.1 读写备份寄存器

第一步：开启PWR和BKP的时钟

第二步：使用PWR，使能BKP和RTC的访问

由于代码太少就不进行封装了。

main.c

```c
#include "stm32f10x.h"   
#include "OLED.h"
#include "Key.h"

uint16_t WriteArry[2]={0x1234,0x5678};//写入数据
uint16_t ReadArry[2];//读出数据
uint8_t KeyNum;//键码值

int main(void)
{
	//初始化OLED、KEY
	OLED_Init();
	Key_Init();

	//初始化BKP
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_PWR,ENABLE);
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_BKP,ENABLE);
	PWR_BackupAccessCmd(ENABLE);

	OLED_ShowString(1,1,"W:");
	OLED_ShowString(2,1,"R:");

	while(1)
	{
		KeyNum=Key_GetNum();//获取键码值
		if(KeyNum==1)//判断键码值
		{
           	//写入BKP数据寄存器
			BKP_WriteBackupRegister(BKP_DR1,WriteArry[0]);
			BKP_WriteBackupRegister(BKP_DR2,WriteArry[1]);
			OLED_ShowHexNum(1,3,WriteArry[0],4);
			OLED_ShowHexNum(1,8,WriteArry[1],4);
            //写入数据自增
			WriteArry[0]++;
			WriteArry[1]++;
		}
        //读取BKP数据寄存器
		ReadArry[0]=BKP_ReadBackupRegister(BKP_DR1);
		ReadArry[1]=BKP_ReadBackupRegister(BKP_DR2);

		OLED_ShowHexNum(2,3,ReadArry[0],4);
		OLED_ShowHexNum(2,8,ReadArry[1],4);
	}
}

```

## 12.5.1 RTC实时时钟

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/12.5.1%20%E7%A8%8B%E5%BA%8F%E6%96%87%E4%BB%B6%E6%A1%86%E5%9B%BE.png?raw=true)

main.c

```c
#include "stm32f10x.h"   
#include "OLED.h"
#include "MyRTC.h"
int main(void)
{
	OLED_Init();
	MyRTC_Init();

	OLED_ShowString(1, 1, "Date:XXXX-XX-XX");
	OLED_ShowString(2, 1, "Time:XX:XX:XX");
	OLED_ShowString(3, 1, "CNT :");
	OLED_ShowString(4, 1, "DIV :");

	while(1)
	{
        //读日期时间
		MyRTC_ReadTime();
        
       	//显示日期
		OLED_ShowNum(1,6,Time[0],4);
		OLED_ShowNum(1,11,Time[1],2);
		OLED_ShowNum(1,14,Time[2],2);
		OLED_ShowNum(2,6,Time[3],2);
		OLED_ShowNum(2,9,Time[4],2);
		OLED_ShowNum(2,12,Time[5],2);
        
        //显示32位计数值和DIV余数寄存器值
		OLED_ShowNum(3,6,RTC_GetCounter(),10);
		OLED_ShowNum(4,6,RTC_GetDivider(),10);
	}
}

```

MyRTC.c

```c
#include "stm32f10x.h" 
#include <time.h>

uint16_t Time[]={2023,1,1,23,59,55};

void MyRTC_SetTime(void)
{
    time_t time_cnt;
    struct tm time_data;

    //将要设定的值传入结构体
    time_data.tm_year = Time[0]-1900;
    time_data.tm_mon = Time[1]-1;
    time_data.tm_mday = Time[2];
    time_data.tm_hour = Time[3];
    time_data.tm_min = Time[4];
    time_data.tm_sec = Time[5];

    //将设定的时间转化为计数值
    time_cnt=mktime(&time_data) - 8*60*60;

    //将计数值写入DIV余数寄存器
    RTC_SetCounter(time_cnt);
    RTC_WaitForLastTask();
}

void MyRTC_Init(void)
{
    //使能BKP和RTC的访问
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_PWR,ENABLE);
    RCC_APB1PeriphClockCmd(RCC_APB1Periph_BKP,ENABLE);
    PWR_BackupAccessCmd(ENABLE);

    if(BKP_ReadBackupRegister(BKP_DR1)!=0xA5A5)
    {
        RCC_LSEConfig(RCC_LSE_ON);//开启LSE晶振
        while(RCC_GetFlagStatus(RCC_FLAG_HSERDY)==RESET);//等待LSE晶振完成标志位
        RCC_RTCCLKConfig(RCC_RTCCLKSource_LSE);//选择LSE时钟源
        RCC_RTCCLKCmd(ENABLE);//开启RTCCLK
            
        //自带while循环所以不需要我们自己判定了
        RTC_WaitForLastTask();//等待上一次写操作完成
        RTC_WaitForSynchro();//等待寄存器同步

        RTC_SetPrescaler(32768-1);//生成1HZ的RTCCLK时钟
        RTC_WaitForLastTask();//等待上一次写操作完成

        MyRTC_SetTime();//设置时间

        BKP_WriteBackupRegister(BKP_DR1,0xA5A5);
    }
    else
    {
        RTC_WaitForLastTask();//等待上一次写操作完成
        RTC_WaitForSynchro();//等待寄存器同步
    }
    
}

void MyRTC_ReadTime(void)
{
    time_t time_cnt;
    struct tm time_data;

    //获取DIV余数寄存器的数值
    time_cnt = RTC_GetCounter() + 8*60*60 ;

    //转化为日期时间
    time_data=*localtime(&time_cnt);
    Time[0]=time_data.tm_year+1900;
    Time[1]=time_data.tm_mon+1;
    Time[2]=time_data.tm_mday;
    Time[3]=time_data.tm_hour;
    Time[4]=time_data.tm_min;
    Time[5]=time_data.tm_sec;
}
```

> 开启晶振后，要调用标志位函数等待标志位。
>
> 每次写操作后，要调用等待写操作完成的函数，函数内部自带while循环不需要我们在自己写while循环。
>
> 初始化时开启晶振、时钟源选择、RTCCLK时钟使能之后记得等待寄存器配置同步完成再进行下一步。
>
> time.h中gmtime函数和localtime函数没有区别，因为stm32不能读取你所在地区，默认都按照伦敦时间来转换了，又因为mktime是转换当地时间的所以我们一般就不用gmtime了都用localtime，想要得到当地时间就加一个偏移就好了。
