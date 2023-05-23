# fstab例子

```
[postgres@localhost clang]$ more /etc/fstab

#
# /etc/fstab
# Created by anaconda on Sun Feb 26 20:15:43 2023
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
/dev/mapper/centos-root /                       xfs     defaults        0 0
UUID=f1679265-c794-43e6-942d-139dbb7dd21f /boot xfs     defaults. 0 0
/dev/mapper/centos-swap swap                    swap    defaults        0 0
```

# fstab字段说明

如果我们需要了解/etc/fstab文件各列字段的说明可以使用man fstab查看并获取帮助。

| 序号 | 列名       | 说明                                                                                                                            |
| :--- | :--------- | :------------------------------------------------------------------------------------------------------------------------------ |
| 1    | fs_spec    | 此列表示要安装的块特殊设备或远程文件系统                                                                                        |
| 2    | fs_file    | 此列表示存储挂载点                                                                                                              |
| 3    | fs_vfstype | 此列表示挂载的文件系统类型                                                                                                      |
| 4    | fs_mntops  | 文件系统挂载参数                                                                                                                |
| 5    | fs_freq    | 是否转储dump，未配置则默认为0，                                                                                                 |
| 6    | fs_passno  | fsck程序使用此字段来确定在重新启动时执行文件系统检查的顺序。0表示不检查；1表示第一位检查，一般用于根挂载点，其他磁盘配置参数2。 |


## fs_spec字段说明

fs_spec字段我们可以使用设备文件名、UUID或者标签，也可以是NFS等远程文件系统。远程文件系统配置方式是`<host>`:`<dir>`，与直接使用mount命令挂载远程文件系统是一致的。其中UUID和LABEL配置方式是LABEL=`<label>` 或者 UUID=`<uuid>`。设备文件、UUID、label作为标识的不同：

使用设备文件名会在当前生效，可能在系统重启之后会出现问题；
label在系统重启之后也会生效，但是要实时注意该设备的label，label是我们在磁盘分区的时候设置的标签；
UUID是分区的唯一标识。
  建议使用UUID的方式挂载。我们可以使用blkid命令查看UUID，可以使用命令ls -l /dev/disk/by-label/查看label。如果磁盘或者分区都没有配置标签，by-label文件可能并不存在。

## fs_file字段说明

  fs_file字段就是我们需要挂载的位置，如果是swap则写swap，其他的写具体的文件系统路径。

## fs_vfstype字段说明

  文件系统类型字段指的是我们需要挂载的文件系统类型，支持的很多ext2, ext3, hfs, hpfs, iso9660, jfs, minix, msdos, ncpfs, nfs, ntfs, proc, qnx4, reiserfs, romfs, smbfs, sysv, tmpfs, udf, ufs, umsdos, vfat, xenix, xfs等，只要是mount命令支持的都可以。linux系统现在常用的是ext4和xfs，也支持window文件系统ntfs、vfat等，如果是交换区，则设置为swap。

## fs_mntops字段说明

  fs_mntops表示挂载时的参数选项，各参数说明如下，我们常用defaults。

* defaults - 使用文件系统的默认挂载参数，例如 ext4 的默认参数为:rw, suid, dev, exec, auto, nouser, async
* auto - 在启动时或键入了 mount -a 命令时自动挂载。
* noauto - 只在你的命令下被挂载。
* exec - 允许执行此分区的二进制文件。
* noexec - 不允许执行此文件系统上的二进制文件。
* ro - 以只读模式挂载文件系统。
* rw - 以读写模式挂载文件系统。
* user - 允许任意用户挂载此文件系统，若无显示定义，隐含启用 noexec, nosuid, nodev 参数。
* users - 允许所有 users 组中的用户挂载文件系统.
* nouser - 只能被 root 挂载。
* owner - 允许设备所有者挂载.
* sync - I/O 同步进行。
* async - I/O 异步进行。
* dev - 解析文件系统上的块特殊设备。
* nodev - 不解析文件系统上的块特殊设备。
* suid - 允许 suid 操作和设定 sgid 位。这一参数通常用于一些特殊任务，使一般用户运行程序时临时提升权限。
* nosuid - 禁止 suid 操作和设定 sgid 位。
* noatime - 不更新文件系统上 inode 访问记录，可以提升性能。
* nodiratime - 不更新文件系统上的目录 inode 访问记录，可以提升性能(参见 atime 参数)。
* relatime - 实时更新 inode access 记录。只有在记录中的访问时间早于当前访问才会被更新。（与 noatime 相似，但不会打断如 mutt 或其它程序探测文件在上次访问后是否被修改的进程。），可以提升性能。
* flush - vfat 的选项，更频繁的刷新数据，复制对话框或进度条在全部数据都写入后才消失。

## fs_freq字段说明

  dump命令将此字段用于这些文件系统，以确定需要转储哪些文件系统。字段可以设置为0或者1，如果第五个字段不存在，返回零值，dump将假设不需要转储文件系统。

## fs_passno字段说明

  fsck程序使用此字段来确定在重新启动时执行文件系统检查的顺序。该字段值可以设置为0|1|2，根文件系统的fs_passno应为1，其他文件系统的fs_passno应为2。将按顺序检查驱动器内的文件系统，但同时检查不同驱动器上的文件系统，以利用硬件中可用的并行性。如果第六个字段不存在或不为零，则返回零值，fsck将假设不需要检查文件系统。
