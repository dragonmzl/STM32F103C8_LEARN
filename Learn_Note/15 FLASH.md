# 15 FLASH

## 15.1.1 FLASH简介

闪存指的是非易失性存储器，掉电不丢失，而本节我们所说的闪存指的是STM32内部的闪存，即程序存储的地方。

> STM32F1系列的FLASH包含程序存储器、系统存储器和选项字节三个部分，通过闪存存储器接口（外设）可以对程序存储器和选项字节进行擦除和编程（出厂设置的Bootloader系统存储器不可更改）
>
> - 读写FLASH的用途：
>   利用程序存储器的剩余空间来保存掉电不丢失的用户数据
>
>   通过在程序中编程（IAP），实现程序的自我更新
>
>   > 通过在程序中写FLASH改变程序本身使得程序自我更新。
>
> - 在线编程（In-Circuit Programming – ICP）用于更新程序存储器的全部内容，它通过JTAG、SWD协议或系统加载程序（Bootloader）下载程序
>
>   > 也就是使用STLINK、串口...等下载
>
> - 在程序中编程（In-Application Programming – IAP）可以使用微控制器支持的任一种通信接口下载程序
>
>   > 自己写一个Booteloader（程序不会覆盖的地方），接收任意通信传来的数据，控制FLASH写数据到程序代码处，完成代码更新，最后跳转到更改出或者直接复位。

### 15.1.2 闪存模块组织

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/15.1.2%20%E9%97%AA%E5%AD%98%E6%A8%A1%E5%9D%97%E7%BB%84%E7%BB%87.png?raw=true)

> - 闪存：主存储器+信息块
> - 闪存存储器接口寄存器其实是一个外设，存放在SRAM里，擦除和编程就通过读写这些寄存器来完成。读取直接使用指针读取就好了不需要接口来。
> - 起始地址后四位是0000、0400、0800、0C00都一定是页的起始地址，可以稍微记一下。
> - FLASH_KEYR键寄存器、FLASH_SR状态寄存器、FLASH_CR控制寄存器...
> - 我们平常说的闪存容量，一般不包括信息块的容量，单指主存储器的容量。

### 15.1.3 FLASH基本结构

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/15.2.2%20FLASH%E8%A7%A3%E9%94%81.png?raw=true)

闪存存储器接口不能对系统存储器进行擦除和编程，选项字节里有很多配置程序存储器的读写保护的，它可以用来排至程序存储器的读写保护。下面将如何操作闪存存储器接口来对程序存储器和选项字节进行擦除和编程。

## 15.2.1 FLASH功能

### 15.2.2 FLASH解锁

 和独立看门狗一样通过写入键寄存器来进行操作，键寄存器能够防止误操作。

> FPEC共有三个键值：
> RDPRT键 = 0x000000A5
> KEY1 = 0x45670123
> KEY2 = 0xCDEF89AB
>
> 解锁：
> 复位后，FPEC被保护，不能写入FLASH_CR
> 在FLASH_KEYR先写入KEY1，再写入KEY2，解锁
> 错误的操作序列会在下次复位前锁死FPEC和FLASH_CR
>
> > 如果有程序先写入KEY2,再写入KEY1就会锁死FPEC和FLASH_CR
>
> 加锁：
> 设置FLASH_CR中的LOCK位锁住FPEC和FLASH_CR

### 15.2.3 使用指针访问存储器

使用指针读指定地址下的存储器：

 uint16_t Data = *((__IO uint16_t *)(0x08000000));

> 读寄存器不需要对FPEC进行解锁

使用指针写指定地址下的存储器：

 *((__IO uint16_t *)(0x08000000)) = 0x1234;

> 写寄存器需要对FPEC进行解锁，如果这里写的是SRAM的地址那就可以直接写入，因为SRAM时可读可写的。

 \#define  __IO  volatile

> 在数据前面加上volatile是一个安全保障措施，在程序逻辑上没作用，***目的是为了防止编译器优化***，keil编译器默认情况下是最低优化等级，这时候加不加volatile都不影响，如果提高优化等级这时候就有问题，比如你用计数函数做一个延时，编译器可能就会直接优化掉，这时候就可以在延时的变量定义前加上volatile，告诉编译器不要优化掉他。编译器还会利用缓存来加速代码，比如要频繁读取某个变量，那么编译器就会把变量转移到高速缓存里，在STM32内核里，有一个类似缓存的工作组寄存器，这些寄存器的访问速度最快，我先把数据放在缓存里，需要读写时，直接访问缓存就好了，用完了再写回内存，假如你有一个中断函数改变了变量的值，但是缓存并不知道，这就导致了数据更新不同步的问题，加上volatile，告诉编译器不要执行缓存优化。
>
> ***总结：如果开启了编译器优化，在无意义加减变量，多线程改变变量读写与硬件相关的存储器时都需要加上volatile，防止编译器优化。***

### 15.2.4 程序存储器编程

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/15.2.4%20%E7%A8%8B%E5%BA%8F%E5%AD%98%E5%82%A8%E5%99%A8%E7%BC%96%E7%A8%8B.png?raw=true)

在写入之前STM32会检查FLASH有没有擦除，如果没有擦除是不会写入的，如果全写0那就会直接写入。

如果你想单独写入一个字节就会比较麻烦，需要把整页数据都读入SRAM里，在SRAM里更改之后，擦除FLASH的那一页，再将更改后的数据写入FLASH。

### 15.2.5 程序存储器页擦除

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/15.2.5%20%E7%A8%8B%E5%BA%8F%E5%AD%98%E5%82%A8%E5%99%A8%E9%A1%B5%E6%93%A6%E9%99%A4.png?raw=true)

写入PER位，就是要执行页擦除

### 15.2.6 程序存储器全擦除

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/15.2.6%20%E7%A8%8B%E5%BA%8F%E5%AD%98%E5%82%A8%E5%99%A8%E5%85%A8%E6%93%A6%E9%99%A4.png?raw=true)

写入MER，就是要执行全擦除

## 15.3.1 选项字节

![](https://github.com/dragonmzl/project-image/blob/main/STM32-LEARN-NOTE-IMAGE/15.3.1%20%E9%80%89%E9%A1%B9%E5%AD%97%E8%8A%82.png?raw=true)

带n的数据意思：例写入USER的值也要相应的写入USER的反码到nUSER，这样的写入参数才是有效的，如果芯片检测到这两个存储器不是反码关系，那代表数据无效，有错误对应的功能就不执行，是一个保护功能，写入反码的过程硬件会自动执行，不用我们操心。库函数更加简单~

> RDP：写入RDPRT键（0x000000A5）后解除读保护，上电默认A5.
>
> USER：配置硬件看门狗和进入停机/待机模式是否产生复位
>
> Data0/1：用户可自定义使用
>
> WRP0/1/2/3：配置写保护，每一个位对应保护4个存储页（中容量），32位保护128页。

### 15.3.2 选项字节编程

1、检查FLASH_SR的BSY位，以确认没有其他正在进行的编程操作

2、解锁FLASH_CR的OPTWRE位

> 也即向FLASH_OPTKEYR按顺序写入KEY1、KEY2

3、设置FLASH_CR的OPTPG位为1

> 表示要选项字节编程

4、写入要编程的半字到指定的地址

5、等待BSY位变为0

6、读出写入的地址并验证数据

### 15.3.3 选项字节擦除

1、检查FLASH_SR的BSY位，以确认没有其他正在进行的闪存操作

2、解锁FLASH_CR的OPTWRE位

> 也即向FLASH_OPTKEYR按顺序写入KEY1、KEY2

3、设置FLASH_CR的OPTER位为1

> 表示要选项字节擦除

4、设置FLASH_CR的STRT位为1

5、等待BSY位变为0

6、读出被擦除的选择字节并做验证

## 15.4.1 器件电子签名

电子签名存放在闪存存储器模块的系统存储区域，包含的芯片识别信息在出厂时编写，不可更改，使用指针读指定地址下的存储器可获取电子签名

- 闪存容量寄存器：

 基地址：0x1FFF F7E0

 大小：16位

- 产品唯一身份标识寄存器：

 基地址： 0x1FFF F7E8

 大小：96位

## 15.5.1 读取内部FLASH

本实验封装了最底层MyFlash.c用来存放底层的读取、擦除、写入FLASH的函数，又封装了上层Store.c用数组（SRAM）来更改FLASH某一页的数据并且每次上电都会把上次断电的数据给读入数组，这样就利用了SRAM的快速随意读写，和ROM的掉电不丢失特性。最后在main.c函数中完成了应用层的函数。

- ***请注意FLASH只能一次写入一个半字！！！！！！！***
- ***库函数实现写入全字的做法是写入两次半字！！！！***

MyFlash.c

```c
#include "stm32f10x.h"     

//利用指针读取一个字的数据
uint32_t MyFlash_ReadWord(uint32_t Adress)
{
    return *((__IO uint32_t*)(Adress));
}

//利用指针读取一个半字的数据
uint16_t MyFlash_ReadHalfWord(uint32_t Adress)
{
    return *((__IO uint16_t*)(Adress));
}

//利用指针读取一个字节的数据
uint8_t MyFlash_ReadByte(uint32_t Adress)
{
    return *((__IO uint8_t*)(Adress));
}

//利用库函数擦除正片FLASH
void MyFlash_EraseAllPages(void)
{
		FLASH_Unlock();
    FLASH_EraseAllPages();
    FLASH_Lock();
}

//利用库函数擦除指定FLASH页
void MyFlash_ErasePage(uint32_t Adress)
{
    FLASH_Unlock();
    FLASH_ErasePage(Adress);
    FLASH_Lock();
}

//利用库函数写入全字
void MyFlash_ProgramWord(uint32_t Adress, uint32_t Data)
{
    FLASH_Unlock();
    FLASH_ProgramWord(Adress,Data);
    FLASH_Lock();
}

//利用库函数写入半字
void MyFlash_ProgramHalfWord(uint32_t Adress, uint16_t Data)
{
    FLASH_Unlock();
    FLASH_ProgramHalfWord(Adress,Data);
    FLASH_Lock();
}
```

Store.c

```c
#include "stm32f10x.h"  
#include "MyFlash.h"

//宏定义 写入的页地址、数组的大小
#define Store_Start_Address 0x0800FC00
#define Store_Count 512

uint16_t Store_Data[Store_Count];

void Store_Init(void)
{
    //在写入的地址的第一个半字定义一个标志位0xA5A5
    if(MyFlash_ReadHalfWord(Store_Start_Address)!=0xA5A5)
    {
        //第一次初始化，写入的地址的第一个半字赋值标志位0xA5A5
        MyFlash_ErasePage(Store_Start_Address);
        MyFlash_ProgramHalfWord(Store_Start_Address,0xA5A5);
        
        //为写入的页全部置0x0000（除标志位）
        for(uint16_t i = 1; i < Store_Count ; i ++)
        {
            MyFlash_ProgramHalfWord(Store_Start_Address+i*2,0x0000);
        }
    }
    
    //每次上电都该页的数据存入Store_Data数组里
    for(uint16_t i = 0; i < Store_Count ; i ++)
    {
        Store_Data[i]=MyFlash_ReadHalfWord(Store_Start_Address+i*2);
    }
    
}

//将数组的数据存入指定页里
void Store_Save(void)
{
    MyFlash_ErasePage(Store_Start_Address);
    for(uint16_t i = 0; i < Store_Count ; i ++)
    {
        MyFlash_ProgramHalfWord(Store_Start_Address+i*2,Store_Data[i]);
    }
}

//将指定页除标志位外全部清0
void Store_Clear(void)
{
    for(uint16_t i = 1; i < Store_Count ; i ++)
    {
        Store_Data[i]=0x0000;
    }
    Store_Save();
}

```

 main.c

```c
#include "OLED.h"
#include "Store.h"
#include "Key.h"

uint16_t KeyNum;

int main(void)
{
	OLED_Init();
	Key_Init();
	Store_Init();

	while (1)
	{
		KeyNum=Key_GetNum();
		if(KeyNum==1)
		{
            //按下按键自增并保存到指定页
			Store_Data[1]++;
			Store_Data[2]++;
			Store_Data[3]++;
			Store_Data[4]++;
			Store_Save();
		}
		else if(KeyNum==2)
		{
            //清除除标志位外指定页的内容
			Store_Clear();
		}
		
        //显示数据
		OLED_ShowHexNum(1,6,Store_Data[0],4);
		OLED_ShowHexNum(2,1,Store_Data[1],4);
		OLED_ShowHexNum(2,6,Store_Data[2],4);
		OLED_ShowHexNum(3,1,Store_Data[3],4);
		OLED_ShowHexNum(3,6,Store_Data[4],4);
	}
}

```

