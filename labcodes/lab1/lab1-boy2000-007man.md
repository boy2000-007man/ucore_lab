练习1：理解通过make生成执行文件的过程
---
1.操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)
```
+ cc kern/init/init.c
+ cc kern/libs/readline.c
+ cc kern/libs/stdio.c
+ cc kern/debug/kdebug.c
+ cc kern/debug/kmonitor.c
+ cc kern/debug/panic.c
+ cc kern/driver/clock.c
+ cc kern/driver/console.c
+ cc kern/driver/intr.c
+ cc kern/driver/picirq.c
+ cc kern/trap/trap.c
+ cc kern/trap/trapentry.S
+ cc kern/trap/vectors.S
+ cc kern/mm/pmm.c
+ cc libs/printfmt.c
+ cc libs/string.c
用以生成所需的可重定位的二进制 (.o) 文件
+ ld bin/kernel
链接之前的二进制文件成ELF格式的可执行文件kernel,即ucore_os
+ cc boot/bootasm.S
+ cc boot/bootmain.c
+ cc tools/sign.c
+ ld bin/bootblock
生成所需.o文件,重链接并生成可执行文件bootblock,组织成启动扇区的格式
dd if=/dev/zero of=bin/ucore.img count=10000
dd if=bin/bootblock of=bin/ucore.img conv=notrunc
dd if=bin/kernel of=bin/ucore.img seek=1 conv=notrunc
新建ucore.img,大小10000*512B,用0填充
先写入bootloader扇区到开头
再写入ucore_os的内容到后续扇区
```
2.一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？
```
硬盘主引导扇区512字节依次由4个部分组成
　　① 主引导程序：是一段程序代码，起到引导系统的关键作用；
　　② 出错信息存放区：引导出错时显示错误信息，主引导程序和出错信息共占446字节，使用各种不同操作系统的分区工具软件或其它分区工具软件所写入的主引导纪录，该部分代码不相同但功能一样；
　　③ 主分区表：硬盘的4个主分区的分区信息都存放在这里，仅仅有64个字节，分为4个分区项；
　　④ 结束标志55AA：主引导记录结束标志，仅为2个字节，是主引导记录是否合法的标志，没有它不行。
```
练习2：使用qemu执行并调试lab1中的软件
---
1.从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。
```
首先运行make debug
在弹出的gdb界面中输入next
注意到在主程序中代码一行行执行,并不进入函数内部
改变输入为step
注意到程序在各个调用的函数语句间不停的跳转,进入了函数内部
```
2.在初始化位置0x7c00设置实地址断点,测试断点正常。
```
在gdb界面中输入b *0x7c00
再输入c继续执行程序
可以看到信息
Breakpoint 2, 0x00007c00 in ?? ()
说明断点设置成功,并且正常暂停
```
3.从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。
```
通过在gdb命令行中设置
define hook-stop
x/i $pc
end
再按前几个问题答案设置断点,单步运行
可得到依次执行的汇编代码为
cli
cld
xor %eax,%eax
mov %eax,%ds
...
经过与bootasm.S和bootblock.asm中的内容对比
可以发现是相匹配的,说明ucore.img在虚拟机中启动时成功加载了bootloader来进一步启动ucore_os
```
4.自己找一个bootloader或内核中的代码位置，设置断点并进行测试。
```
```
练习3：分析bootloader进入保护模式的过程
---
1.为何开启A20，以及如何开启A20
```
A20的关闭是为了实现向下兼容,在32位机上模仿早期16位机对内存地址计算的处理,从而保证BIOS程序对内存的正常访问,而在成功加载BIOS后,为了使得操作系统能充分利用所有的内存地址空间,我们需要开启A20来取消这种模拟行为
开启A20是通过设置8042芯片来实现的
首先等待8042芯片空闲
然后向其控制端口发送写信号
等待8042将写信号处理完毕
接着向其数据端口发送要写的内容,从而将A20开启
```
2.如何初始化GDT表
```
首先生成段描述符表
其中每一段描述符由该段基地址,该段类型,该段操作限制三个参数生成
在完成该表后
记录该表大小与该表起始地址
将记录所在的内存地址使用lgdt指令设置给CPU
从而完成GDT表的初始化
```
3.如何使能和进入保护模式
```
将CPU的cr0寄存器的启用保护位置1,就能使能并让CPU进入保护模式
```
练习4：分析bootloader加载ELF格式的OS的过程
---
1.bootloader如何读取硬盘扇区的？
```
首先读取0x1F7端口数据,返回值不符合要求则重复
向磁盘IO地址寄存器发出读指令与对应的磁盘地址和扇区数
等待磁盘空闲
将0x1F0端口读取到的数据存入指定的内存空间
```
2.bootloader是如何加载ELF格式的OS？
```
首先利用之前从磁盘扇区中读取好的数据
先判断ELF格式正确性
再确定程序头的起始和结束位置
将所有涉及到的段都从磁盘读取到内存中的相应位置
从ELF文件头指出的程序入口地址进入OS
```
练习5：实现函数调用堆栈跟踪函数
---
```
最后一行各数值的含义分别为
ebp指该函数占用内存的栈基指针
eip指调用结束后执行的下一指令的内存地址
四个args指调用发生时栈顶的四个内存单元的内容
```
练习6：完善中断初始化和处理
---
1.中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？
```
占用8个字节
0~15位为入口的段偏移量低16位
16~31位为入口的段选择器
48~63位为入口的段偏移量高16位
```