
# 一、bochs编译

由于Bochs内置的调试器是汇编级的，在操作系统进入内核后，将会有大量的C代码，如果还是使用汇编级的调试还是相当麻烦的，这就需要有源码级的调试了，GDB将成为首选源码级调试器。如果还是Bochs模拟器的话，可以使用GDB连接到Bochs来进行源码级调试。

Bochs默认安装文件是不支持GDB调试的，需要自己重新编译Bochs源码，编译时需要添加参数--enable-gdb-stub

## 1.1 编译带asm调试功能的bochs

```shell
./configure --prefix=/opt/bochs/debug --enable-plugins --enable-debugger --enable-disasm
make
make install
```

## 1.2 编译带gdb调试功能的bochs

```shell
./configure --prefix=/opt/bochs/gdbstub --enable-plugins --enable-disasm --enable-gdb-stub
make
make install

#./configure --prefix=$HOME/bochs --enable-debugger --enable-disasm --enable-iodebug --enable-x86-debugger --with-x --with-x11
```

## 1.3 出错及问题解决

```shell
a.出错: ERROR: X windows gui was selected, but X windows libraries were not found.
 root@ubuntu:~# aptitude install xorg-dev

b.出错: ERROR: pkg-config was not found, or unable to access the gtk+-2.0 package.
 root@ubuntu:~# aptitude install libgtk2.0-dev

c.在configure时同时配了debugger&&gdb-stub会有如下错误./configure --enable-gdb-stub --enable-disasm --enable-debugger
 configure: error: --enable-debugger and --enable-gdb-stub are mutually exclusive
```

## 1.4 gdb+bochs调试linux0.11的总出现缺页中断问题

```c
/*这个问题困扰了好长时间，bxrc配置文件不使能stub的情况下，可以正常运行，但是一进入到gdb中，就总是出现page_fault，查了好久终于发出原因，是bochs在检测到int14 对应的信号量SIGSEGV时会向gdb发送信号量，而gdb接收到此信号默认的处理就是停止，打印，于是在进入到init函数后的执行过程中，如果执行的文件如sh总是出现页面不在物理页面的情况下，就会报错，而此时应该是正常现象。解决方案如下：
    重新安装bochs,修改gdbstub.cc中的*/
    476 if (last_stop_reason == GDBSTUB_EXECUTION_BREAKPOINT ||
    477 last_stop_reason == GDBSTUB_TRACE)
    478 {
    479 write_signal(&buf[1], SIGTRAP);
    480 }
    481 else
    482 {
    483 write_signal(&buf[1], 0);
    484 }
     为
    476 if (last_stop_reason == GDBSTUB_EXECUTION_BREAKPOINT ||
    477 last_stop_reason == GDBSTUB_TRACE)
    478 { 
    479 write_signal(&buf[1], SIGTRAP);
    480 }
    481 else if (last_stop_reason == GDBSTUB_STOP_NO_REASON)
    482 {
    483 write_signal(&buf[1], SIGSEGV);
    484 }
    485 else
    486 {
    487 write_signal(&buf[1], 0);
    488 }
    /*同时在进入gdb时键入：
       handle SIGSEGV nostop noprint ignore 
    这样gdb接收到此信号时将不停止且不打印处理，使得程序得以继续运行。*
```

# 二、bochs配置文件

```c
# 设置bochs在运行中能够使用的内存，我这里设置32MB
mes: 32

# 设置真实机器所对应的BIOS和VGA BIOS
# 这里的路径是按照我上面讲的那两个重要文件的路径来的
# 如果你和我路径不一样，不要直接抄我的！
roimage: file=/home/你的用户名/Downloads/share/bochs/BIOS-bochs-latest
vgaromimage: file=/home/你的用户名/Downloads/share/bochs/VGABIOS-lgpl-latest


# 选择启动磁盘，我这里是选择以硬盘启动
# 如果你想按照软盘启动的话则boot: floppy
boot: disk


# 设置日志文件输出路径
log: bochsout.txt


# 关闭鼠标
mouse: enabled=0


# 磁盘设置
ata0: enabled=1, ioaddr1=0x1f0, ioaddr2=0x3f0, irq=14


# 启动盘信息hd60M.img，下面的hd60M.img为我们启动硬盘
# 我们之后创建的启动硬盘必须放在该路径下且必须同名
# 下面的path也可以写成path="hd60M.img"，不过要求你的hd60M.img在bin文件夹下
ata0-master: type=disk, path="/home/你的用户名/Downloads/bin/./hd60M.img", mode=flat, cylinders=121, heads=16, spt=63


```

# 三、bochs调试指令

## 3.1 help

```shell
[bochs:1](bochs:1) help
h|help - show list of debugger commands
h|help command - show short command description
-*- Debugger control -*-
    help, q|quit|exit, set, instrument, show, trace, trace-reg,
    trace-mem, u|disasm, ldsym, slist
-*- Execution control -*-
    c|cont|continue, s|step, p|n|next, modebp, vmexitbp
-*- Breakpoint management -*-
    vb|vbreak, lb|lbreak, pb|pbreak|b|break, sb, sba, blist,
    bpe, bpd, d|del|delete, watch, unwatch
-*- CPU and memory contents -*-
    x, xp, setpmem, writemem, crc, info,
    r|reg|regs|registers, fp|fpu, mmx, sse, sreg, dreg, creg,
    page, set, ptime, print-stack, ?|calc
-*- Working with bochs param tree -*-
    show "param", restore
```

## 3.2 查看内存中的数据

```shell
[bochs:2](bochs:2) help x
x /nuf `<addr>` - examine memory at linear address
xp /nuf `<addr>` - examine memory at physical address
    nuf is a sequence of numbers (how much values to display)
    and one or more of the [mxduotcsibhwg] format specificators:
    x,d,u,o,t,c,s,i select the format of the output (they stand for hex, decimal, unsigned, octal, binary, char, asciiz, instr)
    b,h,w,g select the size of a data element (for byte, half-word, word and giant word)
    m selects an alternative output format (memory dump)
a. 例如要查看0x102000处的数据-->按16进制4字节打印
[bochs:3](bochs:3) x /8wx 0x102000
0x00102000 <bogus+ 0>:    0x00000007    0x00001007    0x00002007    0x00003007
0x00102010 <bogus+ 16>:    0x00004007    0x00000000    0x00000000    0x00000000
b.例如要查看0x102000处的数据-->按16进制单字节打印
[bochs:4](bochs:4) x /8bx 0x102000
0x00102000 <bogus+ 0>:    0x07    0x00    0x00    0x00    0x07    0x10    0x00    0x00
```

## 3.2 反汇编

```shell
[bochs:5](bochs:5) help u
u|disasm [/count] `<start>` `<end>` - disassemble instructions for given linear address
    Optional 'count' is the number of disassembled instructions
u|disasm switch-mode - switch between Intel and AT&T disassembler syntax
u|disasm hex on/off - control disasm offsets and displacements format
u|disasm size = n - tell debugger what segment size [16|32|64] to use
       when "disassemble" command is used.
a. 例如要反汇编当前指令之后的32条指令
[bochs:6](bochs:6) u /32
00100035: ( ): add eax, 0x00001000 ; 0500100000
0010003a: ( ): cmp edi, 0x00104000 ; 81ff00401000
b. 例如要反汇编0x100000之后的32条指令
[bochs:7](bochs:7) u /32 0x100000
00100000: ( ): cld ; fc
00100001: ( ): mov eax, 0x00000018 ; b818000000

```

## 3.3设断点

```shell
[bochs:8](bochs:8) help b
p|pb|break|pbreak `<addr>` - set a physical address instruction breakpoint
[bochs:9](bochs:9) help lb
lb|lbreak `<addr>` - set a linear address instruction breakpoint
lb 0x100000
b 0x7c00
```
