

## Bosch SMI810 IMU传感器芯片驱动

[TOC]

### 一、总体特点

1.smi8xx家族的传感器分为，陀螺仪+加速度计的组合，分别有1，3，5轴。本次使用的时810，有一轴roll rate(x)陀螺仪和y,z,2轴加速度计。

2.32位数字spi接口通信，16位数据位

3.加速计分为高通和低通，高通可以达到±35g,低通±6g。陀螺仪响应速度范围±300°/s。

4.Yaw rate (Ωz), Roll rate (Ωx)

5.坐标轴图和架构图和pin图，ID pin得上下拉会决定寄存器一位高低值。

![image-20211017133514381](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211017133514381.png)

![image-20211017134429467](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211017134429467.png)

![image-20211017140117077](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211017140117077.png)

### 二、SPI通信

1、spi有两种模式in-frame和out-frame,每种模式分为module commands和sensor commands两种命令。我本次sim8用out-frame。四种时钟和相位都可以用，但是这个手册的时钟极性和平时我所学习的表达相反，比较奇怪。

1.sensor mosi and miso.

![image-20211017134838601](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211017134838601.png)

![image-20211017134859571](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211017134859571.png)

2.module mosi and miso.

![image-20211017134938928](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211017134938928.png)

![image-20211017134951357](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211017134951357.png)

3.相关命令，根据相关命令读取各个通道的数据。

![image-20211017135036676](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211017135036676.png)



![image-20211017135054472](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211017135054472.png)

*4.芯片启动后会先硬件自检，在软件自检和设置，读取spi数据前需要先对EOC位进行值为结束初始化这个过程会有100ms多，所以需要注意，不管你有没有进行芯片初始化设置都要先EOC标志位。*

![image-20211017135707308](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211017135707308.png)

### 三、数据处理

​	1、我们获得的通道数据是一个补码形式，需要转换位原码得到相应的正确值，然后再除以一个响应的比例或者敏感值。比如陀螺仪的敏感值是100，所以我们传感器读到的转换成源码值需要再除以100。

![image-20211017140816509](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211017140816509.png)

![image-20211017140605042](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211017140605042.png)

### 四、寄存器设置和代码编写

1、对x轴陀螺仪传感器各个通道值的读取。

```
	receive_data = 	FUN_HW_AGSPI_BOSCH_SIM810_Read_Fault_v(BOSCH_SMI810_REQREAD_ROLL_RATE);
	ROLL_RATE= (u16)(receive_data >> 4);
```

2、对x轴陀螺仪通道数据进行处理得到数据。

```
	if (((ROLL_RATE >> 15) & 0x01))
		{
			temp_u16 = (~ ROLL_RATE) + 1;
			s16_ROLL_RATE = (s16)(temp_u16 & 0xFFFF);  //Expand 100 times deg/s
			s16_ROLL_RATE *= -1;
		}
		else
		{
			s16_ROLL_RATE = ROLL_RATE;
		}
	S_SPI_last_yawrate_s16 = S_SPI_yawrate_s16;
	S_SPI_yawrate_s16 = s16_ROLL_RATE / 100;

```

