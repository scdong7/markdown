B*树

- 其中叶子节点与分支节点，都是按照顺序排列的（缺省是升序排列，也可以在创建索引时指定为降序排列）
- 如果索引是创建在非分区表上或者索引是分区表上的本地索引的话，则该ROWID占用6个字节；如果索引是创建在分区表上的全局索引的话，则该ROWID占用10个字节。

```c
/*
可用如下方式，转储B*树
alter session set events 'immediate trace name treedump level INDEX_OBJECT_ID';
*/
部分转储内容
----- begin tree dump
   branch: 0x100007c 16777340 (0: nrow: 20, level: 1)
   leaf: 0x100007d 16777341 (-1: nrow: 540 rrow: 540)
   leaf: 0x100007e 16777342 (0: nrow: 533 rrow: 533)
   ……
   leaf: 0x1000189 16777609 (18: nrow: 391 rrow: 391)
----- end tree dump
第一列表示节点类型：branch表示分支节点（包括根节点），而leaf则表示叶子节点.
第二列表示十六进制表示的节点的地址.
第三列表示十进制表示的节点的地址.
第四列表示相对于前一个节点的位置，根节点从0开始计算，其他分支节点和叶子节点从-1开始计算；
第五列的nrow表示当前节点中所含有的索引条目的数量。比如我们可以看到根节点中含有的nrow为20，表示根节点中含有20个索引条目，分别指向9个分支节点；
第六列中的level表示分支节点的层级，对于叶子节点来说level都是0。
第六列中的rrow表示有效的索引条目的数量。
 
/*
根据节点的地址，转换为文件号以及数据块号
select dbms_utility.data_block_address_file(16777340),dbms_utility.data_block_address_block(16777340) from dual;
转储节点块结构alter system dump datafile 4 block 124;
*/
根节点部分转储内容
Branch block dump
=================
header address 218137676=0xd00844c
kdxcolev 1
KDXCOLEV Flags = - - -
kdxcolok 0
kdxcoopc 0x80: opcode=0: iot flags=--- is converted=Y
kdxconco 2
kdxcosdc 1
kdxconro 19
kdxcofbo 66=0x42
kdxcofeo 7884=0x1ecc
kdxcoavs 7818
kdxbrlmc 16777341=0x100007d
kdxbrsno 18
kdxbrbksz 8056 
kdxbr2urrc 0
row#0[8047] dba: 16777342=0x100007e
col 0; len 3; (3):  c2 06 3a
col 1; TERM
row#1[8038] dba: 16777343=0x100007f
col 0; len 3; (3):  c2 0c 0b
col 1; TERM
row#2[8029] dba: 16777344=0x1000080
col 0; len 3; (3):  c2 11 2c
col 1; TERM
……00000000000000000000000000000000000000000000000000000000000
row#18[7884] dba: 16777609=0x1000189
col 0; len 4; (4):  c3 02 01 16
col 1; TERM
----- end of branch block dump -----
End dump data blocks tsn: 4 file#: 4 minblk 124 maxblk 124
kdxcolev表示索引层级号
kdxcolok表示该索引上是否正在发生修改块结构的事务；
kdxcoopc表示内部操作代码；
kdxconco表示索引条目中列的数量；
kdxcosdc表示索引结构发生变化的数量，当你修改表里的某个索引键值时，该值增加；
kdxconro表示当前索引节点中索引条目的数量，但是注意，不包括kdxbrlmc指针；
kdxcofbo表示当前索引节点中可用空间的起始点相对当前块的位移量；
kdxcofeo表示当前索引节点中可用空间的最尾端的相对当前块的位移量；
kdxcoavs表示当前索引块中的可用空间总量，也就是用kdxcofeo减去kdxcofbo得到的。
kdxbrlmc表示分支节点的地址，该分支节点存放了索引键值小于row#0（在转储文档后半部分显示）所含有的最小值的所有节点信息；
kdxbrsno表示最后一个被修改的索引条目号，这里看到是0，表示该索引是新建的索引；
kdxbrbksz表示可用数据块的空间大小。实际从这里已经可以看到，即便是PCTFREE设置为0，也不能用足8192字节。
```

