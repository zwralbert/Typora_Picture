# infineon 高边芯片 BTS54040-LBF 

## 	英飞凌的高边芯片，是BTS系列中的一颗，这个高边开关的是用SPI开控制的，所以记录一下，芯片总体并不麻烦。

[TOC]

###  一.总体特性

1.一颗芯片支持四个高边开关通道，2路输入IO，这两路支持PWM输入。

2.8位spi通信，最大工作频率3MHZ，一般用与汽车12V的高边开关，如照明，指示灯，供暖等，有bulbs和LEDS两种负载控制模式。

3.控制开关电压范围5.5-28V，逻辑电平3.8-5.5V。

4.提供汽车常用的保护和诊断功能。

5.block diagram,结构图如下图所示，主要spi接口和四路输出

![image-20211001131800695](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211001131800695.png)

6.模式，五种模式主要是用到Operative进行控制开关。

![image-20211001132203336](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211001132203336.png)

上电后进入Stand-by 然后寄存器OUT.OUTn给1进入Ready模式，然后DCR.MUX!=111就可以进入正常模式。

### 二.SPI通信

spi通信是8位唯一寄存器，一般和muc的通信注意几个点就行，当然具体的spi通信得看看别人文章学习，不是本文核心，本次我用到的是NXP的汽车级MCU芯片。

1.时钟频率配置位通信频率涉及波特率配置

	  /* DBR =0  BR=8   PBR=5   FP=40MHZ   BUAD= (40/5)*[(1+0)/8]=1M*/
	SPI_2.MODE.CTAR[0].B.PBR=0b10;
	SPI_2.MODE.CTAR[0].B.BR=0b0011;

2.传输数据位的寄存器配置比如我用到的一般是默认16位传输数据位，需要把FMSZ设置八位

		/*frame size*/
	SPI_2.MODE.CTAR[0].B.FMSZ=0b0111;

3.时钟极性和相位要和高边芯片一致才行。BTS54040的时钟极性和相位分为别0,1

	SPI_2.MODE.CTAR[0].B.CPOL=0;   	//高边芯片极性和相位 0 1
	SPI_2.MODE.CTAR[0].B.CPHA=1;

4.读写函数的伪代码

```
u8 FUN_HW_BTS54040_Read(u8 address)
{
	u8 result;
	FUN_HW_Time_Delayus(1);
	SPI2_SendAndGetData_byte(address);
	FUN_HW_Time_Delayus(1);
	result=SPI2_SendAndGetData_byte(address);
	FUN_HW_Time_Delayus(1);
	return result;
}
```

```
void FUN_HW_BTS54040_Write(u8 data)
{
	FUN_HW_Time_Delayus(1);
	SPI2_SendAndGetData_byte(data);
	FUN_HW_Time_Delayus(1);
	SPI2_SendAndGetData_byte(data);
	FUN_HW_Time_Delayus(1);
}
```



### 三.BTS54040寄存器

对于bts54040的寄存器比较简单并不是很多。

1.每次对寄存器进行读写，都会返回状态寄存器。

![image-20211001134450925](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211001134450925.png)

2.初始化芯片，包括Hardconfig，swap模式和DigControl三个寄存器必须得设置。

```
void FUN_HW_BTS54040_Init()
{
	/*LHI set  low*/
	SIUL2.MSCR[PD2].B.SSS = 0;          /*  Source signal is DSPI_0 SOUT  */
	SIUL2.MSCR[PD2].B.OBE = 1;         	/*  OBE=1. */
	SIUL2.MSCR[PD2].B.SRC = 0;       	/*  Full strength slew rate *///
	SIUL2.GPDO[PD2].B.PDO=0;

    BTS54040.HardConfig.R=HWCRCONFIG;		/*hard reset*/
    FUN_HW_BTS54040_Write(BTS54040.HardConfig.R);
    BTS54040.SwapConfig.R=SWCRCONFIG;		/*swag mode 0->output  1->input*/
    FUN_HW_BTS54040_Write(BTS54040.SwapConfig.R);
    BTS54040.DigControl.R=0XF6;			/*diog*/
    FUN_HW_BTS54040_Write(BTS54040.DigControl.R);

}
```

3.高边开关寄存器的开关主要是OUT.OUTn位进行置位，然后读写位设置。

```
void FUN_HW_BTS54040_HSD1_ON(u32 CS)
{

	BTS54040.OutConfig.B.OUT1=1;
	BTS54040.OutConfig.B.WRITE_READ=1;
	FUN_HW_BTS54040_Write(BTS54040.OutConfig.R,CS);

}
```

```
void FUN_HW_BTS54040_HSD1_OFF(u32 CS)
{

	BTS54040.OutConfig.B.OUT1=0;
	BTS54040.OutConfig.B.WRITE_READ=1;
	FUN_HW_BTS54040_Write(BTS54040.OutConfig.R,CS);

}
```

#### 大致就这些，第一次写这么麻烦的归纳文档，肯定有很多不如人意的地方，有问题大家多多指正。