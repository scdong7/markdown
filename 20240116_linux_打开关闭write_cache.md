1. 查看当前硬盘写Cache状态
    hdparm -W  /dev/sda
3. 关闭硬盘的写Cache
    hdparm -W  0 /dev/sda
4. 打开硬盘的写Cache
    hdparm -W  1 /dev/sda
