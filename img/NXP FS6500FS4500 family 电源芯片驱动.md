## NXP FS6500/FS4500 family 电源芯片驱动

### 一、总体综述

1、多电源输出、线性电源电压调节器、开关电源，支持低功耗，支持多种唤醒模式能力和模式的汽车级电源芯片

2、支持1-5v的MCU供电和SMPS(开关模式电源)0.8A,1.5A,2.2A的线性调节器，支持AD,IO,MCU等支持3.3v,5v辅助线性电压监控。

3、多种安全故障监控和检测，反馈故障状态的安全保护。

4、应用于各种BMS，汽车底盘电气系统、汽车动力系统和传动系统等等。

5、16位SPI通信通信，看门狗监控。

5、系列的不同带的功能也不一样，本次用的是FS4505C，支持CANFD。

6、block diagram和pin map

![image-20211103112127953](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211103112127953.png)

![image-20211103112821826](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211103112821826.png)

### 二、工作方式

1、Functional state diagram，芯片工作涉及流程复杂。

芯片大致涉及方面：

1.自检

2.IO配置

3.错误安全machine(过压和欠压)启动和唤醒后启动

4.复位

5.INIT_FS

6.NORMAL WD 喂狗

7.设置复位时间

![image-20211103113122732](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211103113122732.png)

2、 Fail-safe machine state diagram

![image-20211103113254555](C:\Users\曾伟荣\AppData\Roaming\Typora\typora-user-images\image-20211103113254555.png)

3、主要的工作模式就是正常工作和故障状态反馈，其中比较重要的就是芯片的喂狗。

4、芯片喂狗步骤和监控比较麻烦，涉及寄存器也比较多。

大致喂狗步骤涉及：

1.seed

2.LFSR

3.WD refresh counter

4.WD error counter

5.debug 退出需要先接地然后电源复位or唤醒复位。

6.如若喂狗不成功参生一个错误。

7.启动debugging purpose，对DENUG pin设置电平抑制它。

8.复位后进入INIT_FS mode 这个时候spi通信设置watchdog windows refresh.一些寄存器必须这个时候设定不然后面除了这个mode会直接锁定寄存器表有标注。，第一次喂狗前设定好寄存器

9.3种low power模式，1.sleep,所有regulators无用，只有正对安全保障得spi命令有用，三种唤醒方式，1.CAN/LIN,2.I/O input,3.Timer。

###  三、调试总结

1、debug mode，在不知道电源是否能正常喂狗的前提，我们需要进入debug模式下，先启动电源对芯片的正常供电，然后MCU工作正常，能下载程序。debug模式需要接上拉10K左右电阻，不同硬件涉及可能不同。

2、确保SPI通信正常，可以通过测量SPI信号读ID等。spi正常通信下，进行寄存器配置。

3、先对芯片进行正常工作模式初始化，包括寄存器结构体和工作方式中的所有步骤。

```
FS65_RegVal_struct FS65_Registers_InitValues = {
    0x00,	//	INIT_VREG;
    0x40,	//	INIT_WU1;
    0x00,	//	INIT_WU2;->CAN sleep mode 0X20
    0x00,	//	INIT_INT;

    0x00,	//	INIT_INH_INT;
    0xE0,	//	CAN_LIN_MODE; CAN in Normal mode 0XF4 ->shut down lin TX 0xE0
    0x60,	//	INIT_FS1B_TIMING;
    0x00,	//	INIT_SUPERVISOR;
    
    0x10,	//	INIT_FAULT;

   // 0x4D,	//	INIT_FSSM: IO_23 is fail-safe
    0x00,	//	INIT_FSSM: IO_23 not fail-safe
    0x10,	//	INIT_SF_IMPACT;
	0xC0,	//	WD_WINDOW;0x0

    0xB2,	//	WD_LFSR;
    0x00,	//	INIT_WD_CNT;
    0xE0,	//	INIT_VCORE_OVUV_IMPACT;
    0xE0,	//	INIT_VCCA_OVUV_IMPACT;
    
    0xE0,	//	INIT_VAUX_OVUV_IMPACT;

};
```



```
void FS65_Init(void) {

uint32_t fs65_error_code;

// 0. Get silicon version
fs65_error_code = FS65_UpdateRegisterContent(DEVICE_ID_ADR);

// 1.Check LBIST & ABIST1 completion
fs65_error_code = FS65_UpdateRegisterContent(BIST_ADR);
if (fs65_error_code != FS65_RETURN_OK) {
FS65_Error = FS65_SPI_FAIL;
FS65_ErrorCallback();
}

if (INTstruct.BIST.B.LBIST_OK != 1) {
// LBIST Fail, FSx can not be released, user action required
FS65_Error = FS65_LBIST_FAIL;
FS65_ErrorCallback();
}
if (INTstruct.BIST.B.ABIST1_OK !=1) {
// ABIST1 Fail, FSx can not be relaesed, user action required
FS65_Error = FS65_ABIST1_FAIL;
FS65_ErrorCallback();
}

// 2. Get cause of SBC restart
fs65_error_code = FS65_UpdateRegisterContent(INIT_VREG_ADR);
if (fs65_error_code != FS65_RETURN_OK ) {
FS65_Error = FS65_SPI_FAIL;
FS65_ErrorCallback();
}

// 3. Power-On Reset, Init Main registers should be initialized
fs65_error_code = FS65_Init_MSM();
if (fs65_error_code != FS65_RETURN_OK) {
    // Error during initialization, user action required
    FS65_Error = FS65_INIT_MSM_FAIL;
    FS65_ErrorCallback();//ERROR
}



fs65_error_code = FS65_UpdateRegisterContent(WU_SOURCE_ADR); //Read and clear WU sources
if (fs65_error_code != FS65_RETURN_OK ) {
    FS65_Error = FS65_SPI_FAIL;
    FS65_ErrorCallback();
}

// 5. Init FSSM registers
fs65_error_code = FS65_Init_FSSM();
if (fs65_error_code != FS65_RETURN_OK) {
    // Error during initialization, user action required
FS65_Error = FS65_INIT_FSSM_FAIL;
FS65_ErrorCallback();
}

// 5bis; Get current mode of operation
fs65_error_code = FS65_UpdateRegisterContent(MODE_ADR);
if (fs65_error_code != FS65_RETURN_OK) {
    // Error during initialization, user action required
FS65_Error = FS65_INIT_FSSM_FAIL;
FS65_ErrorCallback();
}

// 5ter; Check HW configuration of Vaux/Vcca
fs65_error_code = FS65_UpdateRegisterContent(HW_CONFIG_ADR);
if (fs65_error_code != FS65_RETURN_OK) {
    // Error during initialization, user action required
FS65_Error = FS65_INIT_FSSM_FAIL;
FS65_ErrorCallback();
}

// 6. Read all Diag registers to clear all bits
fs65_error_code = FS65_RETURN_OK;
fs65_error_code += FS65_UpdateRegisterContent(DIAG_VPRE_ADR);
fs65_error_code += FS65_UpdateRegisterContent(DIAG_VCORE_ADR);
fs65_error_code += FS65_UpdateRegisterContent(DIAG_VCCA_ADR);
fs65_error_code += FS65_UpdateRegisterContent(DIAG_VAUX_ADR);
fs65_error_code += FS65_UpdateRegisterContent(DIAG_VSUP_VCAN_ADR);
fs65_error_code += FS65_UpdateRegisterContent(DIAG_CAN_FD_ADR);
fs65_error_code += FS65_UpdateRegisterContent(DIAG_CAN_LIN_ADR);
fs65_error_code += FS65_UpdateRegisterContent(DIAG_SPI_ADR);
fs65_error_code += FS65_UpdateRegisterContent(DIAG_SF_IOS_ADR);
fs65_error_code += FS65_UpdateRegisterContent(DIAG_SF_ERR_ADR);
if (fs65_error_code != FS65_RETURN_OK ) {
FS65_Error = FS65_SPI_FAIL;
FS65_ErrorCallback();
}

// 7. Check if FS1b implemented
fs65_error_code = FS65_UpdateRegisterContent(DEVICE_ID_FS_ADR);
if (fs65_error_code != FS65_RETURN_OK ) {
FS65_Error = FS65_SPI_FAIL;
FS65_ErrorCallback();
}

if (INTstruct.DEVICE_ID_FS.B.FS1 == 1)
{
//FS1B is implemented, it should be zero at this stage
fs65_error_code = FS65_UpdateRegisterContent(RELEASE_FSxB_ADR);
if (fs65_error_code != FS65_RETURN_OK ) {
    FS65_Error = FS65_SPI_FAIL;
    FS65_ErrorCallback();
}
//Read state of FS1B sense
if (INTstruct.RELEASE_FSxB.B.FS1B_SNS == 1)
{
    //FS1B already high
    //fs65_error_code = FS65_UpdateRegisterContent(DIAG_SF_IOS_ADR);
    if (fs65_error_code != FS65_RETURN_OK ) {
	FS65_Error = FS65_SPI_FAIL;
	FS65_ErrorCallback();
    }

    if (INTstruct.DIAG_SF_IOS.B.FS1B_DIAG >= 2)
    {
	//FS1b short-circuited to high, user action required
	FS65_Error = FS65_FS1B_SHORT2HIGH;
	FS65_ErrorCallback();
   }
    else
    {   //FS1B is still high because of the running delay, wait a little
	do {
	    fs65_error_code = FS65_UpdateRegisterContent(RELEASE_FSxB_ADR);
	    if (fs65_error_code != FS65_RETURN_OK ) {
		FS65_Error = FS65_SPI_FAIL;		FS65_ErrorCallback();
	    }
	}
	while (INTstruct.RELEASE_FSxB.B.FS1B_SNS == 1);
   }
}

//Run ABIST2_FS1B
fs65_error_code = FS65_RunABIST2_FS1B();
if (fs65_error_code != FS65_RETURN_OK) {
    // Error during ABIST2, user action required
    FS65_Error = FS65_ABIST2_FS1B_FAIL;
    FS65_ErrorCallback();
}
FUN_HW_Time_Delayus(200); 

fs65_error_code = FS65_UpdateRegisterContent(BIST_ADR);
if (fs65_error_code != FS65_RETURN_OK ) {
    FS65_Error = FS65_SPI_FAIL;
    FS65_ErrorCallback();
}

if (INTstruct.BIST.B.ABIST2_FS1B_OK != 1) {
   // Error during ABIST2, user action required
   FS65_Error = FS65_ABIST2_FS1B_FAIL;
   FS65_ErrorCallback();
}
}

// 8. BIST Vaux if necessary
fs65_error_code = FS65_UpdateRegisterContent(INIT_VAUX_OVUV_IMPACT_ADR);
if (fs65_error_code != FS65_RETURN_OK ) {
FS65_Error = FS65_SPI_FAIL;
FS65_ErrorCallback();
}

if ((INTstruct.INIT_VAUX_OVUV_IMPACT.B.VAUX_FS_OV != 0) && (INTstruct.INIT_VAUX_OVUV_IMPACT.B.VAUX_FS_UV != 0))
{   // Vaux is safety critical, need to BIST it
//Run ABIST2_VAUX
fs65_error_code = FS65_RunABIST2_VAUX();
if (fs65_error_code != FS65_RETURN_OK) {
    // Error during ABIST2 VAUX, user action required
    FS65_Error = FS65_ABIST2_VAUX_FAIL;
    FS65_ErrorCallback();
}

FUN_HW_Time_Delayus(200);  

fs65_error_code = FS65_UpdateRegisterContent(BIST_ADR);
if (fs65_error_code != FS65_RETURN_OK ) {
    FS65_Error = FS65_SPI_FAIL;
    FS65_ErrorCallback();
}

if (INTstruct.BIST.B.ABIST2_VAUX_OK != 1) {
    // Error during ABIST2 VAUX, user action required
    FS65_Error = FS65_ABIST2_VAUX_FAIL;
    FS65_ErrorCallback();
}

  }
}

```

4、喂狗，芯片在初始化时会自带一个256ms窗口的喂狗时间，初始化需要进行喂狗一次，然后进入正常周期喂狗。

```
void FS65_WD_Refresh(void){
    static uint32_t nbWDrefresh = 0;
    static uint32_t FSOUTreleased = 0;
    nbWDrefresh++;

    FS65_UpdateRegisterContent(WD_LFSR_ADR);		//get current LFSR content
    PITstruct.WD_answer = FS65_ComputeLFSR((INTstruct.WD_LFSR.R));
    FS65_RefreshWD((PITstruct.WD_answer));
    FS65_UpdateRegisterContent(DIAG_SF_ERR_ADR);
    FS65_UpdateRegisterContent(DIAG_SF_ERR_ADR);
    FS65_UpdateRegisterContent(WD_COUNTER_ADR);
    /*7th WD clear the FLT_ERR_CNT)*/
    if((FSOUTreleased == 0) & (nbWDrefresh >= 7)){
      FSOUTreleased = 1;
      FS65_ReleaseFS0andFS1out();
    }

//    PIT_0.TIMER[2].TFLG.R=1;
}
```

5、唤醒设置，IO信号唤醒、CAN唤醒等

```
uint32_t FS65_Config_NonInit(void)
{
    uint32_t errorCode = FS65_RETURN_OK;
    FS65_WD_Refresh();
	errorCode += FS65_Set_CAN_LIN_MODE(FS65_Registers_InitValues.CAN_LIN_MODE);
    FS65_UserConfigNonInit();
    /*判断UZ引脚的电平状态，如果为低电平则配置电源管理芯片进入低功率睡眠模式，否则，正常运行*/
    if(POWER_UZ == 0)
	{
    	FS65_SetLPOFFmode();
	}
    return errorCode;
}
```

6、正常喂狗和唤醒设置后，电源能正常工作，接下来就需要对电源上下电调度策略、电压电流问你都采集和故障返回等进行读写诊断和管理，芯片也是刚用上，这一步正在进行中。

