MIPS32处理器需求文档
=====

##一、概述

###项目背景

本项目来源于清华大学计算机科学与技术系计算机组成原理课程、软件工程课程、操作系统课程、编译原理课程联合实验。其目标是设计一颗部分兼容于MIPS32体系结构的CPU，基于FPGA硬件平台实现，并能够运行ucore操作系统。本项目可使学生深入理解计算机系统原理，并在计算机系统级底层开发方面得到训练。

###需求方

- 计算机组成原理课程：刘卫东老师
- 软件工程课程：白晓颖老师
- 操作系统课程：向勇老师
- 编译原理课程：王生原老师

###术语定义

本文档中出现的术语缩写定义如下

术语          | 定义
------------ | -------------
MIPS | 无内部互锁流水线微处理器
CPU  | 中央处理器
ALU  | 算数逻辑单元
MMU  | 内存管理单元
TLB  | 翻译后备缓冲区
RAM  | 随机访问存储器
ROM  | 只读存储器
BIOS  | 基本输入输出系统
Flash | 快闪存储器
CP  | 协处理器

##二、需求描述

###CPU

#####基本功能

CPU需在系统时钟的驱动下，在一个至多个周期内获取并执行一条指令，而后继续获取执行下一条指令，如此往复。支持的指令集为MIPS32指令集的子集，该指令集包含但不限于如下指令:

- 加载、存储指令：LB、LH、LW、SB、SH、SW
- 简单算数运算指令：ADDI、SUB
- 逻辑运算类指令：ANDI、ORI、SLTI、XOR、CLO、SLL、SRA
- 乘除法相关指令：MUL、DIV、MFHI、MTLO
- 分支与跳转指令：J、JAL、JR、BEQ、BGEZ
- 条件移动指令：MOVZ、MOVN
- 异常相关指令：SYSCALL、ERET
- CP0相关指令：MFC0、MTC0
- TLB相关指令：TLBWI

完整的指令集在附录中列出。

为提高运行效率，CPU采用经典5级流水方式，5个阶段分别是取指、译码、执行、访存、回写。取指阶段CPU从指令总线获取一条指令，每次取指在单个时钟周期完成。译码阶段对指令编码进行解释，并获取通用寄存器的值。执行阶段按照指令执行实际运算操作。访存阶段通过数据总线读取或写入内存单元，在个操作在单周期内完成。会写阶段将结果写入通用寄存器。

大部分运算指令在ALU中执行，只消耗一个时钟周期。但乘除法运算过程较复杂，不能在单周期内完成，使用专用的乘除法单元完成运算，多周期运算时暂停流水线。

#####异常处理

由于操作系统的需求，本处理器需要支持必要的异常和中断处理。

处理器需要支持的异常如下：

- Reset：硬件复位
- Interrupt：外部触发中断
- TLBL：无效的加载地址引发异常
- TLBS：无效的存储地址引发异常
- Sys：系统调用指令触发
- RI：无效指令
- Ov：算数运算溢出异常
- AdEL：未对齐的加载地址引发异常
- AdES：未对齐的存储地址引发异常
- TLB Mod：对TLB违规写操作

处理器还需要支持多个中断信号，包括：

- 系统定时器中断：用于操作系统调度
- 串口中断：表示串口收到数据
- 键盘中断：表示键盘收到按键

在流水线设计中，要求支持精确异常处理。即处理器会准确记录发生异常的指令位置（包括位于延迟槽中的指令），并确保异常发生之前的指令均完整执行，之后的指令取消。

中断处理流程参照MIPS32规范。

#####CP0

CP0用于管理硬件，其中包含多个特殊功能寄存器，来配置各项功能。需要实现特殊功能寄存器包括如下几个方面：

- 异常处理：实现一些寄存器，用于保存发生异常时的一些信息，以供异常处理程序使用
- 内存管理：实现用于配置TLB功能的寄存器，与TLB配合完成内存管理
- 功能设置：包括系统定时器、中断使能等功能配置

本项目根据软件需求，只实现MIPS32规范中部分的CP0寄存器和字段。要实现的寄存器和字段在附录中列出。


#####TLB

TLB 是一种全关联的用于转换虚拟地址的结构。TLB 的结构中有若干条目。每一条条目中由两个有机结合的部件构成。这两个部件分别是分别是比较段和物理翻译段。本项目中，需要实现的项目有：

*	比较段
	*	虚拟页编号（VＰÑ）
	*	地址空间标识符（ASID）
	*	全局标志位
*	物理翻译段
	*	物理页帧号（PFN）
	*	合法位
	*	修改位
	*	缓存完整域

###总线

总线系统实现了多主多从的通信，通过仔细设计的交互协议，可至多支持4台主设备和8台从设备进行通信。
访存系统设计了，指令缓存，数据缓存和L2缓存

关于这部分的具体设计文档见Memory System Specification

###串口控制器

串口控制器作为总线上的外设，用于处理兼容RS-232标准的串行数据通讯。控制器固定波特率和比特数，不实现硬件流控功能。该控制器包含一个总线从接口，一路中断信号输出。控制器包括3个特殊功能寄存器，其功能描述如下：

- 状态寄存器：包含但不限于接收非空标志位、发送空标志位，及相应的中断标志位
- 发送数据寄存器：用于存储当前要发送的字节
- 接收数据寄存器：用于存储最新收到的字节

###RAM控制器

RAM控制器分为SRAM和DRAM控制器。可以分别对SRAM和SDRAM进行高效地读写。同时控制器实现了总线的通信协议，可以作为从设备接受主设备的访问指令

###Flash控制器


Flash控制器作为总线上的外设，接受来自CPU的读写请求，并相应地操作Flash芯片。该控制器将Flash空间映射到总线上的一段地址控制，读flash操作可以直接通过总线上读操作完成。flash的擦除与编程操作较复杂，需要软件驱动程序支持，为此设计4个特殊功能寄存器，其功能描述如下：

- 状态寄存器：包含但不限于写操作忙标志
- 写命令寄存器：向flash芯片写入命令字，包括擦除、编程等
- 编程地址寄存器：编程flash时发送的地址
- 编程数据寄存器：编程flash时待写入的数据

###键盘控制器

键盘控制器作为总线上的外设，用于实现PS/2键盘主机端接口。该控制器包含一个总线从接口，一路中断信号输出。控制器包括3个特殊功能寄存器，其功能描述如下：

- 状态寄存器：包含但不限于信号线忙标志位、接收数据有效标志位，及接收数据错误标志位
- 发送数据寄存器：用于存储当前要发送给键盘的控制数据
- 接收数据寄存器：用于存储最新收到的键盘按键

##三、软件环境

本系统设计的目标软件是ucore操作系统。启动操作系统之前，需要首先运行Bootloader，准备必要的系统启动条件。Bootloader固化在FPGA内部ROM中，负责将操作系统代码从Flash拷贝到RAM中，之后跳转到RAM中操作系统入口所在地址，将执行权交给操作系统。Flash控制器在本阶段被使用，但只用到了读取功能。

操作系统初始化过程中，与CPU相关的主要步骤依次为CP0配置、TLB初始化、中断控制器初始化、串口初始化、定时器中断初始化。CPU需要正确地支持这些初始化操作。

之后，在操作系统运行过程中，时钟中断、外设中断和TLB等异常会时常发生，异常处理程序入口有预先放置的代码用于处理异常。

##四、硬件平台

本系统将在真实硬件平台上运行验证，该平台由计算机原理课程实验室提供，技术参数如下：

组件     | 数量   | 型号/参数
--------|-------|----------
FPGA     | 1     | Xilinx<sup>&reg;</sup> Spartan<sup>&reg;</sup>-6 XC6SLX100
SRAM     | 4     | 总共 2M &times; 32bits
Flash    | 1     | 4M &times; 16bits
CPLD     | 1     | Xilinx<sup>&reg;</sup> XC95144XL
串口      | 3     | 
数码管    | 2     |
LED      | 16     |
PS/2接口  | 1     |
拨码开关   | 32    |
按钮开关   | 4     |
以太网控制器| 1     | DM9000A Fast Ethernet Controller
VGA接口    | 1     | 3bits DAC / Channel
USB-OTG控制器 |1    |ISP1362

##五、附录 

###指令集

处理器支持的全部69条指令如下：

Mnemonic	|	Instruction
TLBWI 	|	Write a TLB entry indexed by the Index register

###CP0寄存器

**0 Index**  TLB表入口索引

Index   | 3..0|TLB index. Software writes this field to provide the index to the TLB entry referenced by the TLBR and TLBWI instructions. |R/W|Undefined


	



VPN2 |	31..13 |	VA[31..13] of the virtual address (virtual page number / 2). This field is written by hardware on a TLB exception or on a TLB read, and is written by software before a TLB write. |	R/W 	|Undefined 

Compare|31..0|Interval count compare value|R/W|Undefined

UM 	|4	|If Supervisor Mode is not implemented, this bit denotes the base operating mode of the processor. The encoding of this bit is: 0 Base mode is Kernel Mode; 1 Base mode is User Mode.|	R/W |	Undefined 

BD      | 31   |Indicates whether the last exception taken occurred in a branch delay slot | R |Undefined|
Reserved|	30..16					



1|31|This bit is ignored on write and returns one on read.|R|1
Reserved|11..0

##六、参考文档

1. MIPS32<sup>TM</sup> Architecture For Programmers Volume I: Introduction to the MIPS32<sup>TM</sup> Architecture
2. MIPS32<sup>TM</sup> Architecture For Programmers Volume II: The MIPS32<sup>TM</sup> Instruction Set
3. MIPS32<sup>TM</sup> Architecture For Programmers Volume III: The MIPS32<sup>TM</sup> Privileged Resource Architecture
4. 计算机系统实验准备
5. 计算机系统综合设计与实现——CP0 中断 MMU
6. 基于简化版MIPS32指令集CPU的ucore教学操作系统移植