# 1 CPU基准性能测试

## 1.1 SuperPI测试

SuperPI测试是利用 CPU 的浮点运算能力来计算出π（圆周率），可以用做测试系统稳定性和测试 CPU 计算完后特定位数圆周率所需的时间。

**1、单核测试命令：**

```shell
time echo "scale=5000; 4*a(1)" | bc -l -q &>1c
#其中 scale 指定计算到小数点后多少位。
```

**说明：**
计算原理简单介绍一下，由于 tan（PI/4）=1，所以有 PI=4*atan(1)。
bc 命令是任意精度计算器语言，在 linux 下当做科学计算器使用。-l 指定采用标准数学库，-q 指定 quiet 模式，不打印欢迎语句。

最后用 time 命令输出计算的时间，可以用时间的长短来衡量 CPU 核心的计算能力，比如同样计算到 PI 的小数点后 5000 位，计算时间越短，说明 CPU 核心的计算能力越强。

**2、多核心测试脚本：**

```shell
vi superpi_multicore.sh
#!/bin/bash
cpu_seqs=`cat /proc/cpuinfo | grep proce | sed -e "s/.* //g" | tr -s '\n' ' '`
for cpu_seq in $cpu_seqs;do
time echo "scale=5000; 4*a(1)" | taskset -c $cpu_seq bc -l -q &>1 &
done
```

**数值越小，表明时间越短，说明计算能力越强**

## 1.2 SPECCPU

SPEC是标准性能评估公司（Standard Performance Evaluation Corporation）的简称。SPEC是由计算机厂商、系统集成商、大学、研究机构、咨询等多家公司组成的非营利性组织，这个组织的目标是建立、维护一套用于评估计算机系统的标准。

SPEC CPU是SPEC组织推出的CPU子系统基准测试程序。SPEC CPU的测试程序以源码形式发布，包含多种改进后的实际应用程序，可以运行在多种体系结构之上。SPEC CPU测试中，测试系统的处理器、内存子系统和使用到的编译器都会影响最终的测试性能，而I/O（磁盘）、网络、操作系统和图形子系统对于SPEC CPU 2006的影响非常的小。目前SPEC CPU的最新版本是SPEC CPU 2017,最早的是SPEC CPU 92。

SPEC CPU测试类型包括两种：**整型运算**和 **浮点运算** 。

SPEC CPU 性能测试包括两个部分： **速度测试** ， **吞吐量测试** 。

SPEC CPU把它们分别叫做**speed模式**和 **rate模式** 。其中 **speed是针对单任务的计算速度测试** ， **rate针对系统运行多任务的吞吐量进行测试** 。

# 2 内存基准测试

## 2.1 Stream内存带宽测试

stream由Virginia University提供，生成四种不同模式下的内存读写操作内存带宽表现。

在测试结果中，您会看到一个名为“Triads”的值，该值表示内存带宽。

> **内存带宽 memory bandwidth** 指内存系统在单位时间内的传输数据的能力。
>
> 内存带宽的计算涉及到以下几个因素
>
> 1.内存频率：指内存工作时钟的频率，通常以兆赫兹（MHz）为单位表示。
>
> 2.数据通道数：内存通常有多个数据通道用于传输数据。每个数据通道都能独立传输数据。
>
> 3.数据位宽：指内存每个数据通道传输的数据位数。常见的数据位宽有8，16，32，64位等。
>
> **内存带宽计算公式**：内存带宽=内存频率\*数据通道数\*数据位宽/8
>
> *比如一条DDR3 1333MHz 64bit的内存，理论带宽为：1333**64/8=10664MiB/s = 10.6GiB/s。
>
> 基于计算公式，我们可以得出内存的理论带宽，配合测试工具测试出实际带宽，方便我们查找问题，确认测试结果。

网址：[https://www.cs.virginia.edu/stream/ref.html](https://www.cs.virginia.edu/stream/ref.html)

主要测试内容如下：

| 测试功能        | 操作                  | 解释                                                                                                                                                              | 操作次数      |
| --------------- | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| **Copy**  | c[i]+a[i]             | 复制操作，即从内存单元中读取一个数，并复制到其他内存单元中，两次访问内存操作                                                                                      | 1R1W 一读一写 |
| **Scale** | b[i]=scalar*c[i]      | 乘法操作，即从内存单元中读取一个数，与常数相乘，得到的记过存到其他内存单元，两次访问内存操作                                                                      | 1R1W 一读一写 |
| **Add**   | c[i]=a[i]+b[i]        | 加法操作，从两个内存单元中分别读取两个数，将其进行加法操作后，得到的结果写入另一个内存单元中，<br />3次访问内存操作                                               | 2R1W 二读一写 |
| **Triad** | a[i]=b[i]+scalar*c[i] | 是前面三种的结合，先从内存中读取一个数，与一个常数相乘得到一个乘积，<br />然后从另一个内存单元中读取一个数与刚才乘积结果相加，得到的结果写入内存。3次访问内存操作 | 2R1W 二读一写 |

编译参数如下：

```
# 参数介绍
# -mtune=native -march=native 针对CPU指令的优化，此处由于编译机即运行机器。故采用native的优化方法。
# -O3 编译器编译优化级别。
# -mcmodel=medium 当单个Memory Array Size 大于2GB时需要设置此参数（小于2GB时设置无效）。
# -fopenmp 适应多处理器环境，开启后，程序默认线程为CPU线程数。也可以在运行前设置进程数
# 设置方法: export OMP_NUM_THREADS=x  x为你想设置的线程数
# -DSTREAM_ARRAY_SIZE=100000000：这个参数是对测试结果影响最大，也是最需要关注的一个参数，指定计算中a[],b[],c[]数组的大小。
# -DNTIMES=40：执行的次数，并且从这些结果中选最优值
```

单线程编译：

```
gcc -mtune=native -march=native -O3 -mcmodel=medium -DSTREAM_ARRAY_SIZE=200000000 -DNTIMES=30 stream.c -o stream.o
```

多线程编译（一般采用多线程编译）：

```
gcc -mtune=native -march=native -O3 -mcmodel=medium -fopenmp -DSTREAM_ARRAY_SIZE=200000000 -DNTIMES=30 stream.c -o stream.o
```

执行结果：

```shell
[root@localhost stream]# ./stream.o
-------------------------------------------------------------
STREAM version $Revision: 5.10 $
-------------------------------------------------------------
This system uses 8 bytes per array element.
-------------------------------------------------------------
Array size = 2000000 (elements), Offset = 0 (elements)
Memory per array = 15.3 MiB (= 0.0 GiB).
Total memory required = 45.8 MiB (= 0.0 GiB).
Each kernel will be executed 30 times.
 The *best* time for each kernel (excluding the first iteration)
 will be used to compute the reported bandwidth.
-------------------------------------------------------------
Number of Threads requested = 1
Number of Threads counted = 1
-------------------------------------------------------------
Your clock granularity/precision appears to be 1 microseconds.
Each test below will take on the order of 6397 microseconds.
   (= 6397 clock ticks)
Increase the size of the arrays if this shows that
you are not getting at least 20 clock ticks per test.
-------------------------------------------------------------
WARNING -- The above is only a rough guideline.
For best results, please be sure you know the
precision of your system timer.
-------------------------------------------------------------
Function    Best Rate MB/s  Avg time     Min time     Max time
Copy:           16079.8     0.002255     0.001990     0.003259
Scale:          12116.8     0.002878     0.002641     0.003381
Add:            13393.2     0.004018     0.003584     0.004968
Triad:          13219.9     0.004340     0.003631     0.008490
-------------------------------------------------------------
Solution Validates: avg error less than 1.000000e-13 on all three arrays
-------------------------------------------------------------
[root@localhost stream]#
```

## 2.2 MLC内存延迟测试

mlc 为 intel 的内存测试工具，可以有效方便的测试内存延时。

**步骤 1 下载**

```shell
wget https://software.intel.com/content/dam/develop/external/us/en/protected/mlc_v3.9.tgz
```

**步骤 2 安装**

```shell
#解压安装：
tar -xzvf mlc_v3.9.tgz
```

**步骤 3 测试步骤**

```shell
./Linux/mlc --idle_latency -e -r -l128 -D8192
```

# 3 IO基准测试

## 3.1 IPOS(Input/Output Operations Per Second)

每秒进行读写操作的次数

以如下为例计算IPOS：

* 转速：7200 rpm(转/分)
* 平均延迟：4.17ms
* 随机读取寻道时间：11ms

平均延迟表示旋转延迟时间，它的值等于旋转时间的1/2。 即 60/7200*1000/2=4.17ms

寻道次数因磁盘容量、质量等其他因素不同而不同。

IPOS是对寻道延迟和旋转延迟两种操作的平均时间的测量。

如上例：IPOS=1000/（11+4.17）=65.9

> **当磁头靠近磁盘的外部时，读取速度比内侧更快**
>
> **短行程技术意思是指使用磁盘中最快的部分**

## 3.2 dd命令吞吐量

```shell
time sh -c "dd if=/dev/zero of=test.txt bs=8k count=blocks && sync " time dd if=test.txt of=/dev/null bs=8k
#blocks 建议用实际内存2大小的文件估算数据块数，文件足够大，以至于系统无法在内存中缓存所有读写操作，以减少缓存对测试结果的影响。
```

## 3.3 其他测试工具

Bonnie++、iozone、fio等
