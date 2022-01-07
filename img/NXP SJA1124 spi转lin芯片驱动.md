## NXP SJA1124 spi转lin芯片驱动	

[TOC]

### 一、总体特点

1.sja1124 是lin控制器和收发器集成一体芯片，支持4路lin通道。

2.支持LIN 2.0, LIN 2.1, LIN 2.2, LIN 2.2A和相关协议国际标准,最快可以达到20kb波特率.

3.通过spi或者lin可以在low power模式下唤醒

4.需要外接时钟作为lin通信时钟源。

5.硬件lin通信，只需对寄存器进行配置。

6.结构框架图

![image-20211004184306540](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211004184306540.png)



### 二、工作模式

1.芯片的五种模式如下图，注意手册分为芯片模式和lin模式，会有重名的模式，如果涉及到lin的模式会加上lin，不然一般为芯片模式。

![image-20211004184406459](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211004184406459.png)

2.芯片上电后会有3.2s的INITI idie timeout，没有在这个时间进行spi通信或者lin唤醒，芯片进入low power模式。

3.lin模式有lin sleep，lin normal,lin initializional。这几种是lin的模式，比如初始化模式，如果你用几路lin的话，每一路lin都需要单独初始化。

### 三、spi通信

spi通信是8位寄存器，一般和muc的通信注意以下几个点就行，当然具体的spi通信得看看别人文章学习，不是本文核心，本次我用到的是NXP的汽车级MCU芯片。

1.时钟频率配置位通信频率涉及波特率配置

	  /* DBR =0  BR=8   PBR=5   FP=40MHZ   BUAD= (40/5)*[(1+0)/8]=1M*/
	SPI_2.MODE.CTAR[0].B.PBR=0b10;
	SPI_2.MODE.CTAR[0].B.BR=0b0011;

2.传输数据位的寄存器配置比如我用到的一般是默认16位传输数据位，需要把FMSZ设置八位

		/*frame size*/
	SPI_2.MODE.CTAR[0].B.FMSZ=0b0111;

3.时钟极性和相位要和高边芯片一致才行。SJA1124的时钟极性和相位分为别0,1

	SPI_2.MODE.CTAR[0].B.CPOL=0;   	//lin芯片极性和相位 0 1
	SPI_2.MODE.CTAR[0].B.CPHA=1;

4.读写函数的伪代码

```
u8 FUN_HW_SJA1124_ReadData(u8 address)
{
	u8 data;

	HW_SJA1124_CS1=0;
	FUN_HW_Time_Delayus(1);
	SPI3_SendAndGetData_byte(address,HW_SJA1124_CS1);
	SPI3_SendAndGetData_byte(SJA1124READ,HW_SJA1124_CS1);		//读8位长数据
	data=SPI3_SendAndGetData_byte(address,HW_SJA1124_CS1);
	FUN_HW_Time_Delayus(1);
	HW_SJA1124_CS1=1;

	return data;
}

```

```
void FUN_HW_SJA1124_WriteData(u8 address,u8 data)
{

	HW_SJA1124_CS1=0;
	FUN_HW_Time_Delayus(1);
	SPI3_SendAndGetData_byte(address,HW_SJA1124_CS1);
	SPI3_SendAndGetData_byte(SJA1124WRITE,HW_SJA1124_CS1);		//读8位长数据
	SPI3_SendAndGetData_byte(data,HW_SJA1124_CS1);
	FUN_HW_Time_Delayus(1);
	HW_SJA1124_CS1=1;

}
```

### 三、lin通信

1.通信分为主机和从机，本次芯片lin只能作为主机，简单的描述一下lin通信协议，具体的通信学习还是得去看专业得文章，不是本次重点，本次lin通信是硬件lin,不需要软件模拟通信协议。

2.lin通信主要分为报头（header）和回复响应（response）两个部分.如下图

![image-20211004190740479](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211004190740479.png)

3.lin逻辑电平分为显性(0)和隐性(1)两种。lin是一主多从通信方式，从机节点一般最多可以到16个，通信过程必须主机先发header,接着从机response,主机在response,然后无限response。

4.报头分别有间隔场，同步场和标识符场。间隔场一般为大于13个显性电平bit,而同步场是固定为“0x55"一个字节，标识符则是发送header的PID，PID可以分为ID和奇偶校验，ID范围为0-0x3f也就是64，不过很多特殊ID不能用，而这里的奇偶有特别的校验算法可以查表。

5.响应场就分为数据场和校验和场，数据场可以是2/4/8字节，一般用8个字节和can差不多。校验场就校验和场是数据场所有字节和的反码。现在一般分为经典型(classic)和增强型(enhanced)。



### 四、芯片寄存器配置

1.寄存器也和模式一样，分别芯片寄存器和lin寄存器。

2.首先需对芯片配置寄存器进行初始化，涉及IO、工作模式、时钟中断寄存器配置。

```
void FUN_HW_SJA1124_Init(void)
{
	//引脚初始化
	SIUL2.MSCR[PC15].B.SSS = 0;			/* Pin functionality as GPIO */
	SIUL2.MSCR[PC15].B.OBE = 1;          /* Output Buffer Enable off */
	SIUL2.MSCR[PC15].B.IBE = 0;			/* Input Buffer Enable on */
	/*clk*/
	SIUL2.MSCR[PB6].B.SSS = 1;			/* Pin functionality as GPIO */
	SIUL2.MSCR[PB6].B.OBE = 1;          /* Output Buffer Enable off */
	
	/*模式选择*/
	SJA1124.MODE_reg.B.RST=1;//复位
	SJA1124.MODE_reg.B.LPMODE=0;    //0.正常模式 1.低功耗
	FUN_HW_SJA1124_WriteData(SJA1124MODE,SJA1124.MODE_reg.R);
	/*时钟频率设置*/
	SJA1124.PLLCFG_reg.B.PLLMULT=0X8;//  CLK=32M
	FUN_HW_SJA1124_WriteData(PLLCFG,SJA1124.PLLCFG_reg.R);
	//默认开启唤醒WUIE中断
	//开启所有中断
	SJA1124.INT1EN_reg.R=0x0F;
	FUN_HW_SJA1124_WriteData(INT1EN,SJA1124.INT1EN_reg.R);
	SJA1124.INT2EN_reg.R=0x3E;
	FUN_HW_SJA1124_WriteData(INT2EN,SJA1124.INT2EN_reg.R);
	SJA1124.INT3EN_reg.R=0xFF;
	FUN_HW_SJA1124_WriteData(INT3EN,SJA1124.INT3EN_reg.R);
}
```

3.需要对LIN通道初始化，必须在INIT=1即LIN初始化模式下配置相应的寄存器，主要内容也就是lin通信的检验方式，间隔长度，相关寄存器配置，还有波特率和中断。

```
void FUN_HW_SJA1124_LIN1Init(void)
{

	/*LIN1 configuration */
	SJA1124.LCFG1_reg[0].B.CCD=0;//硬件检验
	SJA1124.LCFG1_reg[0].B.MBL=0X3;//break显性电平长度 16位->13位 官方demo
	SJA1124.LCFG1_reg[0].B.SLEEP=0;//正常模式
	SJA1124.LCFG1_reg[0].B.INIT=1;//初始化模式配置必须再初始化模式下
	FUN_HW_SJA1124_WriteData(LCFG1LIN1,SJA1124.LCFG1_reg[0].R);
	/*2-bit delimiter*/
	SJA1124.LCFG2_reg[0].B.TBDE=1;
	SJA1124.LCFG2_reg[0].B.IOBE=1;
	FUN_HW_SJA1124_WriteData(LCFG2LIN1,SJA1124.LCFG2_reg[0].R);
	
	/*idle on timeout*/
	SJA1124.LITC_reg[0].B.IOT=1;
	FUN_HW_SJA1124_WriteData(LITCLIN1,SJA1124.LITC_reg[0].R);
	/*2 stop bit congfiguraton*/
	SJA1124.LGC_reg[0].B.STOP=1;
	FUN_HW_SJA1124_WriteData(LGCLIN1,SJA1124.LGC_reg[0].R);
	/*response time*/
	SJA1124.LRTC_reg[0].R=0X0E;//响应时间待定设置的是推荐值
	FUN_HW_SJA1124_WriteData(LRTCLIN1,SJA1124.LRTC_reg[0].R);
	
	/*baud rate  若CLK为16M */
	SJA1124.LFR_reg[0].B.FBR=0X2;//设置为3
	FUN_HW_SJA1124_WriteData(LFRLIN1,SJA1124.LFR_reg[0].R);
	SJA1124.LBRM_reg[0].B.IBR=0X00;//不用高位
	FUN_HW_SJA1124_WriteData(LBRMLIN1,SJA1124.LBRM_reg[0].R);
	SJA1124.LBRL_reg[0].B.IBR=LIN1BAUD;//低位104  104*16+3=1667    32M/1667=19200;
	FUN_HW_SJA1124_WriteData(LBRLLIN1,SJA1124.LBRL_reg[0].R);
	
	SJA1124.LCFG1_reg[0].B.INIT=0;//设置回正常模式
	FUN_HW_SJA1124_WriteData(LCFG1LIN1,SJA1124.LCFG1_reg[0].R);
	/*interrupt */
	FUN_HW_SJA1124_WriteData(LIELIN1,0xF7);//使能所有中断
}
```

4.接着是lin通信的设置，包括ID，数据。

```
void FUN_HW_SJA1124_LIN1SendFrame( Lin1_PduType* LIN1Frame)
{
	//
	SJA1124.LC_reg[0].B.HTRQ=1;
	FUN_HW_SJA1124_WriteData(LIN1LC,SJA1124.LC_reg[0].R);
//	SJA1124.LCOM2_reg.B.L1HTRQ=1;
//	FUN_HW_SJA1124_WriteData(LIN1LC,SJA1124.LCOM2_reg.R);
	SJA1124.LBI_reg[0].B.ID=LIN1Frame->lin1id;//ID标识符设定范围0-0x3f
	FUN_HW_SJA1124_WriteData(LIN1LBI,SJA1124.LBI_reg[0].R);
	SJA1124.LINLBC_reg[0].B.DFL=LIN1Frame->lin1dfl;//字节个数  DFL=value-1
	SJA1124.LINLBC_reg[0].B.DIR=LIN1Frame->lin1dir;//
	SJA1124.LINLBC_reg[0].B.CCS=LIN1Frame->lin1ccs;//0为lin2.0 enhaned 校验版本 1为lin1.3
	FUN_HW_SJA1124_WriteData(LIN1LBC,SJA1124.LINLBC_reg[0].R);


	/*tx data*/
	FUN_HW_SJA1124_WriteData(LIN1LDB1,LIN1Frame->lin1data[0]);
	FUN_HW_SJA1124_WriteData(LIN1LDB2,LIN1Frame->lin1data[1]);
	FUN_HW_SJA1124_WriteData(LIN1LDB3,LIN1Frame->lin1data[2]);
	FUN_HW_SJA1124_WriteData(LIN1LDB4,LIN1Frame->lin1data[3]);
	FUN_HW_SJA1124_WriteData(LIN1LDB5,LIN1Frame->lin1data[4]);
	FUN_HW_SJA1124_WriteData(LIN1LDB6,LIN1Frame->lin1data[5]);
	FUN_HW_SJA1124_WriteData(LIN1LDB7,LIN1Frame->lin1data[6]);
	FUN_HW_SJA1124_WriteData(LIN1LDB8,LIN1Frame->lin1data[7]);


	SJA1124.LS_tag[0].R=FUN_HW_SJA1124_ReadData(LIN1STATU);

}
```

5.对header和response的结构体设计，通信函数放入结构体就OK。

```
typedef struct
{
  /* LBI register. */
  uint8_t lin1id;                /* LIN frame identifier. */
  /* LBC register. */
  uint8_t lin1dfl;               /* Data field length (number of bytes - 1). */
  Lin_FrameResponseType lin1dir; /* Response type (master or slave response). */
  Lin_FrameCsModelType lin1ccs;  /* Checksum model type (classic or enhanced checksum calculation). */
  /* LCF register. */
  uint8_t lin1cf;                /* Checksum to be transmitted in case checksum calculation is disabled. */
  /* LBDx registers. */
  uint8_t lin1data[8];           /* Pointer to data to send. */
} Lin1_PduType;
```

```
/* LIN1 Frame configuration. */
Lin1_PduType Frame1 = {
  .lin1id = 0x20,                /* Identifier - ID. */
  .lin1dfl = 7,                  /* Data field length = number of data bytes - 1. */
  .lin1dir = LIN_MASTER_RESPONSE, /* Direction - master to slave. */
  .lin1ccs = LIN_ENHANCED_CS,     /* Enhanced checksum. */
  .lin1cf = 0x40,                /* Checksum - not needed, checksum calculated automatically. */
  .lin1data = { 0x11, 0x02, 0x31, 0x40, 0x65, 0x36, 0x78, 0x08 }, /* Data to send. */
};
```



lin通信刚刚接触不久，还有很多bug还要慢慢解决。

