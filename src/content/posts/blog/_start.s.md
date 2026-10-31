---
title: 番外：F401启动文件解析
published: 2026-10-29
tags:
  - 嵌入式
category: 内存
draft: false
author: 刘相凡
description: 番外1
series: 内存
seriesOrder: 9
---
基于`startup_f401xx.s`
# 整体结构
## 栈配置
```asm
Stack_Size      EQU     0x00000400

                AREA    STACK, NOINIT, READWRITE, ALIGN=3
Stack_Mem       SPACE   Stack_Size
__initial_sp
```
+ `Stack_Size EQU 0x00000400`：栈大小为 `0x400 = 1024` 字节，即 1KB
+ `AREA STACK, NOINIT, READWRITE, ALIGN=3`：定义一个名为 `STACK` 的段：
	+ `NOINIT`：未初始化
	+ `READWRITE`：可读可写
	+ `ALIGN=3`：按8字节对齐
+ `Stack_Mem SPACE Stack_Size`：保留 1KB 栈空间
	+ Stack_Mem：标号，栈空间的起始地址（栈底）
	+ SPACE：伪指令，**预留一块未初始化的内存**
+ `__initial_sp`：位于栈空间之后，表示栈顶地址。Cortex-M 的栈是向下增长的，所以复位后初始 SP 设置为这个地址
## 堆配置
```asm
Heap_Size       EQU     0x00000200

                AREA    HEAP, NOINIT, READWRITE, ALIGN=3
__heap_base
Heap_Mem        SPACE   Heap_Size
__heap_limit
```
- 堆大小为 `0x200 = 512` 字节。
- `__heap_base` 是堆起始地址。
- `__heap_limit` 是堆结束地址。
- 堆主要用于 `malloc()`、`free()` 等动态内存分配。裸机程序如果不使用动态内存，也可以不关心它。
>为什么堆需要`__heap_base`标记起始地址，栈不需要？
>	因为堆和栈的使用方式不同，C库需要的信息不同
>	堆需要告诉C库堆从哪里开始、哪里结束，动态内存分配函数`malloc()`等也需要知道一块可用的连续内存空间，所以要有起始和结束的边界
>	栈只需要知道**栈顶指针**，因为Cortex-M的栈**向下增长**，栈底其实也有符号 `Stack_Mem`，只是它不像堆那样需要单独叫 `__stack_base`
## 代码属性设置
```asm
                PRESERVE8
                THUMB
```
- `PRESERVE8`：要求栈保持 8 字节对齐，符合 ARM 架构要求。
- `THUMB`：后续代码使用 Thumb 指令集。Cortex-M4 只支持 Thumb/Thumb-2 指令
>栈段已经有 `ALIGN=3`，为什么还要 `PRESERVE8`?
>	`ALIGN=3`只能保证`STACK`段的起始地址是8字节对齐的
>	`PRESERVE8`保证在程序运行中，栈指针是9字节对齐的
## 中断向量表
```asm
                AREA    RESET, DATA, READONLY
                EXPORT  __Vectors
                EXPORT  __Vectors_End
                EXPORT  __Vectors_Size
```
+ 定义只读数据段`RESET`，其中放置中断向量表
```asm
__Vectors       DCD     __initial_sp               ; Top of Stack
                DCD     Reset_Handler              ; Reset Handler
                DCD     NMI_Handler                ; NMI Handler
                DCD     HardFault_Handler          ; Hard Fault Handler
                ...
```
向量表每一项是4字节地址，以此为：
1. 第 0 项：初始栈指针 `__initial_sp`；
2. 第 1 项：复位处理函数 `Reset_Handler`；
3. 后面是系统异常：
    - NMI
    - HardFault
    - MemManage
    - BusFault
    - UsageFault
    - SVC
    - DebugMon
    - PendSV
    - SysTick 
4. 再后面是 STM32F401 的外设中断，例如：
    - `WWDG_IRQHandler`
    - `PVD_IRQHandler`
    - `EXTI0_IRQHandler`
    - `DMA1_Stream0_IRQHandler`
    - `USART1_IRQHandler`
    - `SPI1_IRQHandler`
    - `I2C1_EV_IRQHandler`
    - `OTG_FS_IRQHandler`
    - `FPU_IRQHandler`
    - `SPI4_IRQHandler` 等
5. `DCD 0`表示保留项，不指向任何处理函数
```asm
__Vectors_End

__Vectors_Size  EQU  __Vectors_End - __Vectors
```
- `__Vectors_End`：向量表结束位置。
- `__Vectors_Size`：向量表总大小，等于结束地址减起始地址
> 启动文件只是“定义”向量表。向量表最终放在Flash起始地址还是被重映射，由链接脚本和芯片启动模式决定。
+ 向量表中的字的内容是符号地址，链接器最终会把这些符号地址解析为真实地址，链接脚本决定`RESET`段最终放在哪里
> 中断向量表中每个位置放的符号是固定的吗？
> 	中断向量表中符号的位置由架构和芯片定义决定，不能随意调换。如果表项顺序写错，中断就会跳转到错位的函数。
> 	即使用户重新定义了函数，覆盖的只是同名函数的地址，位置不能变，因为硬件只关心“函数地址”，它只会在相应中断发生时，去对应位置取中断ISR的地址
## Reset_Handler
```asm
Reset_Handler    PROC
                 EXPORT  Reset_Handler             [WEAK]
        IMPORT  SystemInit
        IMPORT  __main

                 LDR     R0, =SystemInit
                 BLX     R0
                 LDR     R0, =__main
                 BX      R0
                 ENDP
```
+ 芯片复位流程
## 默认异常处理函数
```asm
NMI_Handler     PROC
                EXPORT  NMI_Handler                [WEAK]
                B       .
                ENDP
```
+ 类似地，`HardFault_Handler`、`MemManage_Handler`、`BusFault_Handler`、`UsageFault_Handler`、`SVC_Handler`、`DebugMon_Handler`、`PendSV_Handler`、`SysTick_Handler` 都定义为弱符号
+ 默认实现`B .`，即跳转到当前指令，形成死循环
>为什么异常处理函数默认要死循环？
>	保留现场，方便调试
## 默认外设中断处理函数
```asm
Default_Handler PROC

                EXPORT  WWDG_IRQHandler                   [WEAK]
                EXPORT  PVD_IRQHandler                    [WEAK]
                ...
                EXPORT  SPI4_IRQHandler                   [WEAK]

WWDG_IRQHandler
PVD_IRQHandler
...
SPI4_IRQHandler

                B       .

                ENDP
```
默认死循环：
+ 如果用户没有实现某个中断函数，但是该中断被触发，程序会停在默认死循环中，方便调试
>用户自己实现函数，中断发生时还是通过向量表跳转吗？
>	是的，硬件永远只认向量表
## 用户栈和堆初始化
```asm
                 IF      :DEF:__MICROLIB

                 EXPORT  __initial_sp
                 EXPORT  __heap_base
                 EXPORT  __heap_limit

                 ELSE

                 IMPORT  __use_two_region_memory
                 EXPORT  __user_initial_stackheap

__user_initial_stackheap

                 LDR     R0, =  Heap_Mem
                 LDR     R1, =(Stack_Mem + Stack_Size)
                 LDR     R2, = (Heap_Mem +  Heap_Size)
                 LDR     R3, = Stack_Mem
                 BX      LR

                 ALIGN

                 ENDIF
```
+ 这是用于配合C库的堆栈初始化
# 理解
```text
定义 1KB 栈、512B 堆
定义中断向量表
向量表第 0 项 = 初始栈顶
向量表第 1 项 = Reset_Handler
Reset_Handler 调用 SystemInit()
Reset_Handler 跳转 __main
__main 初始化 C 运行环境并调用 main()
所有未实现的中断默认进入死循环
用户可在 C 文件中重定义同名中断函数覆盖弱定义
```
+ 启动文件是芯片复位后到`mian()`之间的桥梁，也是中断向量表的提供者
## 堆栈开辟
**启动文件开头跟结尾的堆栈配置有什么却别跟联系呢？**
+ 开头：定义物理内存区域
	+ 在目标文件里真正预留栈空间和堆空间
	+ 给这些空间打上边界符号
	+ 链接器最终把这些段放到RAM里
+ 末尾：向C库报告堆栈边界，这一段不再分配内存，而是：
	+ 如果使用MicroLIB：导出边界符号，Micro LIB自己会使用它们
	+ 如果使用标准C库：实现 `__user_initial_stackheap` 函数，把堆栈边界通过 R0～R3 返回给 C 库
+ 联系：它们两个指向同一块内存

|对比项|开头 `STACK`/`HEAP` 段|末尾 `__user_initial_stackheap`|
|---|---|---|
|本质|物理内存段定义|C 库接口函数|
|是否分配内存|是，真正预留|否，只返回已有地址|
|发生阶段|编译/链接期|运行期，`__main` 初始化时|
|面向对象|链接器、芯片 RAM|C 库|
|使用条件|始终存在|标准库需要；MicroLIB 不需要|
|关键符号|`Stack_Mem`、`__initial_sp`、`__heap_base`、`__heap_limit`|返回 R0～R3|
|关系|定义内存实体|报告同一块内存的边界|
