## MEMSIC  MXP7205VF两轴加速度计传感器芯片驱动

[TOC]



### 一、总体特性

1.芯片本身并不复杂，是两轴加速度计，常用于汽车电子稳定系统。

2.16位spi通信，最大速度到8MHZ，芯片又分为10位和14位获取数据模式。

3.自带滤波器，硬件设计的滤波直接输出相对稳定的数值但也又偏差。

### 二、spi通信

1.spi通信主要是读取MXP7205的数值，本次我用到的是14位模式，从下图中而本身传感器值直接是可以通过SPI读取，直接发送读x，y轴的MSB和LSB的值相与得到xy轴的值，而spi时钟相位和极性都位0。

![image-20211005151005114](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211005151005114.png)

2.芯片返回的值如下图，主要在于低10位返回的数值。

![](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211005151713107.png)

### 三、读数据

1.从上面SPI通信的图片可以看出，写命令高四位和低四位决定你读取的值的通道。然后返回的低10位是我们得到的数据值。

2.本次是14位数据模式，所以是把读到的x或者y高10位向左移4位，再把读到x或y低4位与高10位两者做或运算就得到一个14位数据就得到x或y通道的数值。实际值x,y的值需要读到的数值在除以8。

```
void MXP7205_Getxy(void)
{

	u16 xl,xh,yl,yh;
	s16 x,y;
	s16 accx,accy;
	yh = QSPI3_SendAndGetData(0x2000);	//Accelerometer output LSB register, x-channel
	xl = QSPI3_SendAndGetData(0x2001);	//Accelerometer output LSB register, y/z-channel
	yl = QSPI3_SendAndGetData(0x2002);	//Accelerometer output MSB register, x-channel
	xh = QSPI3_SendAndGetData(0x2003);	//Accelerometer output MSB register, y/z-channel
	x = (s16)(xh&0x03FF)<<4 | (xl&0x000F);	//14bits operation mode,
	y = (s16)(yh&0x03FF)<<4 | (yl&0x000F);	//14bits operation mode,
	accx = (s16)x/8; //14bits operation mode, 800LSB/g.
	accy = (s16)y/8; //14bits operation mode, 800LSB/g.

}
```

