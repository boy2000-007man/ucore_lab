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