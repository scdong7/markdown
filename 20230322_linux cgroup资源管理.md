# 1. cgroup介绍

    cgroup是control group的简称，最初由Google工程师提出，后来编进linux内核。
    它为Linux内核提供了一种任务聚集和划分的机制，通过一组参数集合将一些任务组织成一个或多个子系统。

* cgroups是实现IaaS虚拟化(kvm、lxc等)，PaaS容器(Docker等)的资源管理控制部分的底层基础
* 子系统是根据cgroup对任务的划分功能将任务按照一种指定的属性划分成的一个组，主要用来实现资源的控制。
* 在cgroup中，划分成的任务组以层次结构的形式组织，多个子系统形成一个数据结构中类似多根树的结构。
* cgroup包含了多个孤立的子系统，每一个子系统代表单一的资源，

> 目前：redhat默认支持10个子系统，但默认只挂载了8个子系统，
>
> ubuntu 12.04 默认支持8个子系统，但默认只挂载了5个子系统。

# 2. cgroup子系统介绍

> ps: 当然也用户可以自定义子系统并进行挂载。

下面对每一个子系统进行简单的介绍：

| 系统       | 用途                                                                                                                                                                                            |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| blkio      | 设置限制每个块设备的输入输出控制。例如:磁盘，光盘以及usb等等。                                                                                                                                  |
| cpu        | 使用调度程序为cgroup任务提供cpu的访问。                                                                                                                                                         |
| cpuacct    | 产生cgroup任务的cpu资源报告。                                                                                                                                                                   |
| cpuset     | 如果是多核心的cpu，这个子系统会为cgroup任务分配单独的cpu和内存。                                                                                                                                |
| devices    | 允许或拒绝cgroup任务对设备的访问。                                                                                                                                                              |
| freezer    | 暂停和恢复cgroup任务。                                                                                                                                                                          |
| memory     | 设置每个cgroup的内存限制以及产生内存资源报告。                                                                                                                                                  |
| net_cls    | 标记每个网络包以供cgroup方便使用。                                                                                                                                                              |
| ns         | 名称空间子系统。                                                                                                                                                                                |
| perf_event | 增加了对每group的监测跟踪的能力，即可以监测属于某个特定的group的所有线程以及             运行在特定CPU上的线程，<br />此功能对于监测整个group非常有用，具体参见 http://lwn.net/Articles/421574/ |

# 3. libcgroup工具安装

以Centos 6.4为例

```shell
yum install libcgroup
service cgconfig start   #开启cgroups服务
chkconfig cgconfig on   #开机启动
[root@localhost /]# ls /cgroup/
blkio  cpu  cpuacct  cpuset  devices  freezer  memory  net_cls
```

cgroup启动时，会读取配置文件/etc/cgconfig.conf的内容，根据其内容创建和挂载指定的cgroup子系统。

# 4. cgroup配置文件分析

## 4.1 cgconfig.conf

    /etc/cgconfig.conf是cgroup配置工具libcgroup用来进行cgroup组的定义，参数设定以及挂载点定义的配置文件，

主要由mount和group两个section构成。

### 4.1.1 mount section的语法格式如下：

```shell
mount {
    `<controller>` = `<path>`;
    ...
}
#########################################
# controller：内核子系统的名称
# path：该子系统的挂载点
#########################################
举个例子：
mount {
    cpuset = /cgroup/red;
}

#上面定义相当于如下shell指令：
mkdir /cgroup/red
mount -t cgroup -o cpuset red /cgroup/red
```

### 4.1.2 group section的语法格式如下：group `<name>` {

```shell
    [`<permissions>`]
    `<controller>` {
        `<param name>` = `<param value>`;
        …
    }
    …
}
################################################################################
## name: 指定cgroup的名称
## permissions：可选项，指定cgroup对应的挂载点文件系统的权限，root用户拥有所有权限。
## controller：子系统的名称
## param name 和 param value：子系统的属性及其属性值
#################################################################################
举个例子：
mount {  ## 定义需要创建的cgroup子系统及其挂载点，这里创建cpu与cpuacct（统计）两个cgroup子系统
	cpu = /mnt/cgroups/cpu;
	cpuacct = /mnt/cgroups/cpu;
}
group daemons/www { ## 定义daemons/www(web服务器进程)组
	perm {      ## 定义这个组的权限
		task {
			uid = root;
			gid = webmaster;
                }
                admin {
                	uid = root;
                        gid = root;
                }
            }
        cpu {      ## 定义cpu子系统的属性及其值，即属于词组的任务的权重为1000
        	cpu.shares = 1000;
            }
}
group daemons/ftp { ## 定义daemons/ftp(ftp进程)组
	perm {
		task {
			uid = root;
			gid = ftpmaster;
                     }
		admin {
		uid = root;
		gid = root;
		}
	}
        cpu {  ## 定义词组的任务的权重为500
        	cpu.shares = 500;
        }
}
#上面配置文件定义相当于执行了如下shell命令：
mkdir /mnt/cgroups/cpu
mount -t cgroup -o cpu,cpuacct cpu /mnt/cgroups/cpu
mkdir /mnt/cgroups/cpu/daemons
mkdir /mnt/cgroups/cpu/daemons/www
chown root:root /mnt/cgroups/cpu/daemons/www/*
chown root:webmaster /mnt/cgroups/cpu/daemons/www/tasks
echo 1000 > /mnt/cgroups/cpu/daemons/www/cpu.shares
mkdir /mnt/cgroups/cpu/daemons/ftp
chown root:root /mnt/cgroups/cpu/daemons/ftp/*
chown root:ftpmaster /mnt/cgroups/cpu/daemons/ftp/tasks
echo 500 > /mnt/cgroups/cpu/daemons/ftp/cpu.shares
```

## 4.2 cgrules.conf

```shell
[postgres@cent6 Desktop]$ cat /etc/cgrules.conf
# /etc/cgrules.conf
#
#Each line describes a rule for a user in the forms:
#
#<user>            <controllers>        <destination>
#<user>:<process name>    <controllers>        <destination>
#
#Where:
# <user> can be:
#        - an user name
#        - a group name, with @group syntax
#        - the wildcard *, for any user or group.
#        - The %, which is equivalent to "ditto". This is useful for
#          multiline rules where different cgroups need to be specified
#          for various hierarchies for a single user.
#
# <process name> is optional and it can be:
#     - a process name
#     - a full command path of a process
#
# <controller> can be:
#      - comma separated controller names (no spaces)
#      - * (for all mounted controllers)
#
# <destination> can be:
#      - path with-in the controller hierarchy (ex. pgrp1/gid1/uid1)
#
# Note:
# - It currently has rules based on uids, gids and process name.
#
# - Don't put overlapping rules. First rule which matches the criteria
#   will be executed.
#
# - Multiline rules can be specified for specifying different cgroups
#   for multiple hierarchies. In the example below, user "peter" has
#   specified 2 line rule. First line says put peter's task in test1/
#   dir for "cpu" controller and second line says put peter's tasks in
#   test2/ dir for memory controller. Make a note of "%" sign in second line.
#   This is an indication that it is continuation of previous rule.
#
#
#<user>      <controllers>      <destination>
#
#john          cpu        usergroup/faculty/john/
#john:cp       cpu        usergroup/faculty/john/cp
#@student      cpu,memory    usergroup/student/
#peter           cpu        test1/
#%           memory        test2/
#@root            *        admingroup/
#*        *        default/
# End of file
 postgres      memory           test1/
#
[postgres@cent6 Desktop]$
```

# 5. cgroups资源控制

## 5.1 cgroups管理进程cpu资源

1. 跑一个耗cpu的脚本

```shell
x=0
while [ True ];do
    x=$x+1
done;
```

2. top可以看到这个脚本基本占了100%的cpu资源

```shell
 PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND      
30142 root      20   0  104m 2520 1024 R 99.7  0.1  14:38.97 sh
```

3. 下面用cgroups控制这个进程的cpu资源

```shell
mkdir -p /cgroup/cpu/foo/   #新建一个控制组foo
echo 50000 > /cgroup/cpu/foo/cpu.cfs_quota_us  #将cpu.cfs_quota_us设为50000，相对于cpu.cfs_period_us的100000是50%
echo 30142 > /cgroup/cpu/foo/tasks
```

4. 然后top的实时统计数据如下，cpu占用率将近50%，看来cgroups关于cpu的控制起了效果

```shell
PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND 
30142 root      20   0  105m 2884 1024 R 49.4  0.2  23:32.53 sh 
```

5. cpu控制组foo下面还有其他的控制，还可以做更多其他的关于cpu的控制

```shell
[root@localhost ~]# ls /cgroup/cpu/foo/
cgroup.event_control  cgroup.procs  cpu.cfs_period_us  cpu.cfs_quota_us  cpu.rt_period_us  cpu.rt_runtime_us  cpu.shares  cpu.stat  notify_on_release  tasks
```

## 5.2 cgroups管理进程内存资源

1. 跑一个耗内存的脚本，内存不断增长

```shell
x="a"
while [ True ];do
    x=$x$x
done;
```

2. top看内存占用稳步上升

```shell
  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND    
30215 root      20   0  871m 501m 1036 R 99.8 26.7   0:38.69 sh  
30215 root      20   0 1639m 721m 1036 R 98.7 38.4   1:03.99 sh 
30215 root      20   0 1639m 929m 1036 R 98.6 49.5   1:13.73 sh
```

3. 下面用cgroups控制这个进程的内存资源

```shell
mkdir -p /cgroup/memory/foo
echo 1048576 >  /cgroup/memory/foo/memory.limit_in_bytes   #分配1MB的内存给这个控制组
echo 30215 > /cgroup/memory/foo/tasks

[root@localhost ~]# sh /home/test.sh 
已杀死
```

> 发现之前的脚本被kill掉,因为这是强硬的限制内存，当进程试图占用的内存超过了cgroups的限制，会触发out of memory，导致进程被kill掉。实际情况中对进程的内存使用会有一个预估，然后会给这个进程的限制超配50%比如，除非发生内存泄露等异常情况，才会因为cgroups的限制被kill掉。也可以通过配置关掉cgroups oom kill进程，通过memory.oom_control来实现（oom_kill_disable 1），但是尽管进程不会被直接杀死，但进程也进入了休眠状态，无法继续执行，仍让无法服务。

4. 关于内存的控制，还有以下配置文件，关于虚拟内存的控制，以及权值比重式的内存控制等

```shell
[root@localhost /]# ls /cgroup/memory/foo/
cgroup.event_control  memory.force_empty         memory.memsw.failcnt         
memory.memsw.usage_in_bytes      memory.soft_limit_in_bytes  memory.usage_in_bytes  tasks
cgroup.procs          memory.limit_in_bytes      memory.memsw.limit_in_bytes  
memory.move_charge_at_immigrate  memory.stat                 memory.use_hierarchy
memory.failcnt        memory.max_usage_in_bytes  memory.memsw.max_usage_in_bytes  
memory.oom_control               memory.swappiness           notify_on_release
```

## 5.3 cgroups管理进程io资源

1. 跑一个耗io的脚本

```shell
 dd if=/dev/sda of=/dev/null &
```

2. 通过iotop看io占用情况，磁盘速度到了284M/s

```shell
30252 be/4 root      284.71 M/s    0.00 B/s  0.00 %  0.00 % dd if=/dev/sda of=/dev/null  
```

3. 下面用cgroups控制这个进程的io资源

```shell
mkdir -p /cgroup/blkio/foo

echo '8:0   1048576' >  /cgroup/blkio/foo/blkio.throttle.read_bps_device
#8:0对应主设备号和副设备号，可以通过ls -l /dev/sda查看
echo 30252 > /cgroup/blkio/foo/tasks
```

4. 再通过iotop看，确实将读速度降到了1M/s

```shell
30252 be/4 root      993.36 K/s    0.00 B/s  0.00 %  0.00 % dd if=/dev/sda of=/dev/null  
```

5. 对于io还有很多其他可以控制层面和方式，如下

```shell
[root@localhost ~]# ls /cgroup/blkio/foo/
blkio.io_merged         blkio.io_serviced      blkio.reset_stats                blkio.throttle.io_serviced       blkio.throttle.write_bps_device   blkio.weight          cgroup.procs
blkio.io_queued         blkio.io_service_time  blkio.sectors                    blkio.throttle.read_bps_device   blkio.throttle.write_iops_device  blkio.weight_device   notify_on_release
blkio.io_service_bytes  blkio.io_wait_time     blkio.throttle.io_service_bytes  blkio.throttle.read_iops_device  blkio.time                        cgroup.event_control  tasks
```
