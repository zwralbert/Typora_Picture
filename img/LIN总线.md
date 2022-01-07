## lin总线通信

[TOC]



#### 一、综述

1.采用**单主多从**的组网方式，无CAN总线那样的仲裁机制，最多可连接**16个节点（1主15从）**。

2.主要用于can总线的协助辅助功能，汽车低速反应要求应用，对硬件要求简单，仅需UART/SCI 接口，辅以简单驱动程序便可实现 LIN 协议。故几乎所有的MCU均支持LIN。

![image-20211220141436851](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211220141436851.png)

3.低成本，最大传输速率20kbps。通常低速设计2400bps,中速设计9600bps,高速设计19200bps.

5.在LIN的标准中，令牌被称为“header”，数据被称为“response”，报文被称为“Frame”。在“header”中含有表示报文身份的“ID”，各个节点根据“ID”决定是否发送“Response”。同时，LIN报文是地址寻址方式，总线上的所有节点都能收到报文。

![image-20211220142601209](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211220142601209.png)



### 二、LIN 报文结构

1、LIN 总线上具有“显性”和“隐性”两种互补的逻辑电平。其中，**显性电平（参考地电压）是逻辑 0，隐性电平（电源电压）是逻辑1。**

2、如上介绍所说LIN采用的是“主从”通信方式。LIN报文的一帧由“Header”和“Response”组成，“Header”由主任务（主节点）发送，“Response”由从任务（主节点或者从节点）发送。下面将分别介绍“Header”和“Response”。

**Header**

“Header”又可以分为“Break”、“Synch”和“PID”3个场（图1）。

![img](https://pic2.zhimg.com/80/v2-7d18c17f771a7582b4503c4668322ca5_1440w.jpg)

​								图1　header 结构 

**Break场**

Break场不同与其他场，它有意的造成UART通信中的FramingError（从起始位到第10位没有检出停止位时的错误）来提示LIN总线中的所有从节点之后要开始进行LIN报文的传输了。

Break场又可以分为“Break”，“Break-delimiter”，“Break”为13位以上的显性位，“Break-delimiter”为1位的隐形位，“Break-delimiter”是“Break”结束的标志。



**Synch场**

Synch场即同步场，第一讲在介绍同期信号时提到过同步场。同步场是为了修正各个节点间时钟的误差，固定发送0x55的UART数据（包含起始位/停止位）。从节点根据最初和最终的下降沿除以8来算出1位的时间，并以此作为基准来调整自己的时钟误差（图 2）。如果从节点使用的是高精度时钟的话（允许误差±1.5%），则不需要调整时钟的误差。



![img](https://pic4.zhimg.com/80/v2-6d83bea5c67488d6aa432ac335a8b2ef_1440w.jpg)



图2synch结构 （参照VectorJapan资料作成）

**PID场**

ID范围 0-0x3f

PID场标识LIN报文识别信息，由6位（位0~位5）的报文ID和2位（位6、位7）的奇偶校验位，合计8位组成（图3）。



![img](https://pic4.zhimg.com/80/v2-6aabe2b39ac35c2768c65ef4b122bd57_1440w.jpg)







**Response**

Response由“数据”和“和校验”2个场组成。都可以通过UART的形式进行传输。



![img](https://pic1.zhimg.com/80/v2-e6a9208293266d235944ed0b01d4d160_1440w.jpg)



图4数据场结构 （参照VectorJapan资料作成）

**数据场**

数据场最大可以传输8Byte数据



**和校验场**

和校验即我们通常说的Checksum，用来确认接收的数据是否正常。和校验场的具体值为各个数据场的和的反数，如果有溢出的话，则需要取余运算（mod256）。和校验有“标准和校验（Classic Checksum）”和“扩张和校验（Enhanced Checksum）”两种形式：

​	lin1.3 classic checksum lin2.0 enhanced checksum

标准和校验

计算对象为所有数据场

LIN1.x为所有报文都使用

LIN2.x为诊断报文（ID60~61）使用



扩张和校验

计算对象为PID场和数据场

LIN2.x为报文ID0~59使用



通过上述结构，各个报文在LIN总线上传输。通过Header调整时钟误差，确认报文信息，进行数据的接收和发送，并且有奇偶校验与和校验来确保数据的正确性。



**时间规定**

LIN报文的传输是根据LIN的时间表执行的。按照LIN的硬件结构，报文的传输时间可以分为“Response时间”和“间隔时间”，设计时间表时需要考虑两者的误差（图5）。



![img](https://pic4.zhimg.com/80/v2-2ab9772bec606c0284d4fee98e563703_1440w.jpg)

### 三、总线传输

1、主机节点报文发送

**A.主机任务主要执行以下功能：**

1.定义总线上的**通信速率**。（同步场？待考）

2.**发送报文帧头**，包含同步间隔场、同步场和标识符场三个部分。

3.监控总线通信，通过**校验和**确定数据正确性与否。

4.使从机进入**唤醒或睡眠状态**，并响应从机的唤醒要求。

![image-20211220133739076](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211220133739076.png)

2、从机节点报文发送

**B.从机任务既可运行于主机又可运行于从机，它主要完成以下功能：**

1.等待主机任务发送的同步间隔，使从机与主机于同步场中获得同步。

2.分析标识符场，若与自己相关，则接收或发送数据，若与自己无关则什么都不做。

3.检查和发送**校验和。**

4.接受主机任务的唤醒和睡眠请求。

![image-20211220133618871](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211220133618871.png)

### 四、诊断方式

**主节点的诊断：**

主节点通常与CAN等主干网络连接。因此，不使用LIN而是使用主干网络进行诊断。

**从节点的诊断：**

LIN通信由主节点进行通信控制，因此从节点不能与诊断测试仪直接通信。所以，从节点的诊断必须通过主节点进行。

[LIN通信入门二 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/114693797)

[LIN总线入门 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/38833752)

[LIN总线协议简介_IOT2017的博客-CSDN博客_lin协议](https://blog.csdn.net/IOT2017/article/details/86008184)
