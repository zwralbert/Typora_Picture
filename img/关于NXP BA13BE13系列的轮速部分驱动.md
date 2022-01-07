## 关于NXP 汽车ABS ASIC芯片BA13系列轮速部分驱动

[TOC]

### 一、BA13芯片介绍

1、BA13是专门用于汽车ABS系统设计的一款芯片，但也可以拓展到EPS/ESC上的配套芯片驱动

2、8个低边开关分别4个PWM和4个电流阀门控制，高边保护开关，四个轮速信号接口，两个车速输出接口，500HZ的电机PWM控制

3、16位SPI通信可以达到10MHZ，通信高边驱动开关，K线接口，警告灯接口

4、看门狗设置，各种安全诊断信息保护和监控以及警告，

5、block diagram

![image-20211114121908447](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211114121908447.png)

### 二、轮速部分介绍

1、四个轮速传感器可以是一型霍尔的轮速传感器，可以是标准二型传感器或者智能二型传感器或者智能三型传感器，传感器一段接地一端接入BA13轮速通道接口，BA13会把传感器输入的电流信号转化为电压脉冲信号，同时会有监控保护，提供电流检测、过压过流保护、开路短路检测、和过热保护。

![image-20211114122320879](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211114122320879.png)

2、三种不同传感器的区别，一型的霍尔传感器应该是一种方波信号输出，二型传感器应该是以占空比不同的矩形波信号，三型信号则是幅值和占空比都不一样的脉冲信号。

![image-20211114123150487](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211114123150487.png)



3、轮速相关寄存器，message 2为轮速类型选着寄存器，选择相应的类型的轮速传感器。message 4 为轮速传感器的漏电流安全追踪开启与否。messaga 5轮速信号通道打开与关闭。message20-23为类型二和三的传感器数据读取，主要是方向数据，类型2的比较简单读取，类型三的需要Manchester解码结果读取需要了解曼彻斯特码。

![image-20211114192432933](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211114192432933.png)

![image-20211114194159709](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211114194159709.png)

![image-20211114194257897](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211114194257897.png)

![image-20211114194419221](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211114194419221.png)

### 三、spi通信

1、spi时钟和相位为0，0。在速度比较高的情况下主要各种延时设置尽量长一点保证数据不出错。

2、写的一帧16位包括15bit的齐偶校验，BA13采取偶校验方式。然后14-10bit为寄存器ID，ID为message号，然后SED为看们狗喂狗值。每一帧地址不同得具体看。

![image-20211114195403786](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211114195403786.png)

3、读数据每一个message不一样，需要具体去看了。比如这1号message读到的值就是ASIC ba13的芯片的一些状态。

![image-20211114195800348](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211114195800348.png)

