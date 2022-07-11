# REDO 简介

REDO 日志文件是 Oracle 数据库实例恢复机制中最为关键的组成部分。REDO 日志机制的目 的就是确保数据库实例或者服务器发生故障时，不会导致数据库崩溃，不会丢失已经提交的数据。 Oracle 为了实现这个目标，将数据库的变化数据记录在 REDO 日志文件中，而且通过 Redo-Write-Ahead(RWA)机制确保数据库被变更之前，其变更的 REDO 日志信息必须先写入日 志缓冲区，而事务提交之前也必须首先将日志缓冲中和这个事务相关的 REDO 信息写入 REDO 日志文件。

## Log-Force-at-Commit

在事务 提交时，和这个事务相关的 REDO 日志数据，包括 COMMIT 记录，都必须从日志缓冲区中写入 REDO 日志文件，此时事务提交成功的信号才能发送给用户进程。通过这种机制，哪怕已经提交 的事务中的部分缓冲缓存还没有被写入数据文件就发生了实例故障，在做实例恢复的时候，也可 以通过 REDO 日志的信息将不一致的数据前滚。

## Write-Ahead-Log

日志写入优先包含两方面的算法。

1. 在某个缓冲区缓存的修改的变化矢量还没有写入 REDO 日志文件之 前，这个修改后的缓冲区的数据不允许被写入数据文件，这样就确保了在数据文件中不可能包含 未在 REDO 日志文件中记录的变化。

2. 在对某个数据的 UNDO 信息的变化矢量没 有被写入 REDO 日志之前，这个缓冲区的修改不能被写入数据文件。

## 改变向量(Change Vector)

改变向量表示对数据库某一个数据块所做的一次变更。CV中包含了变更数据块的版本号、事务操作代码、变更从属数据块的地址(dba)以及跟新后的数据。

CV 的定义里包含了两层含义，即一个 CV 只针对一个数据块的变更， 一个 CV 只包含一个变化。每个 CV 都包含了对文件的修改，因此在每个 CV 中都有一个 OPCODE 指出修改的类型。不同 OPCODE 的 CV，其组成也是不同的，OPCODE 的取值范围如代码清单

```c
   Layer 1 : Transaction Control - KCOCOTCT     
				Opcode 1 : KTZFMT 
				Opcode 2 : KTZRDH 
				Opcode 3 : KTZARC
				Opcode 4 : KTZREP
    Layer 2 : Transaction Read -  KCOCOTRD     
     
    Layer 3 : Transaction Update -  KCOCOTUP     
     
    Layer 4 : Transaction Block -  KCOCOTBK     [ktbcts.h]
         Opcode 1 : Block Cleanout 
         Opcode 2 : Physical Cleanout 
         Opcode 3 : Single Array Change
         Opcode 4 : Multiple Changes to an Array
         Opcode 5 : Format Block
    Layer 5 : Transaction Undo -  KCOCOTUN     [ktucts.h]
         Opcode 1 : Undo block or undo segment header - KTURDB
         Opcode 2 : Update rollback segment header - KTURDH
         Opcode 3 : Rollout a transaction begin 
         Opcode 4 : Commit transaction (transaction table update) 
- no undo record 
         Opcode 5 : Create rollback segment (format) - no undo record 
         Opcode 6 : Rollback record index in an undo block - KTUIRB
         Opcode 7 : Begin transaction (transaction table update) 
         Opcode 8 : Mark transaction as dead 
         Opcode 9 : Undo routine to rollback the extend of a rollback segment 
         Opcode 10 :Redo to perform the rollback of extend of rollback segment 
                    to the segment header. 
         Opcode 11 :Rollback DBA in transaction table entry - KTUBRB 
         Opcode 12 :Change transaction state (in transaction table entry) 
         Opcode 13 :Convert rollback segment format (V6 -> V7) 
         Opcode 14 :Change extent allocation parameters in a rollback segment 
         Opcode 15 :
         Opcode 16 :
         Opcode 17 :
         Opcode 18 :
         Opcode 19 : Transaction start audit log record
         Opcode 20 : Transaction continue audit log record     
         Opcode 24 : Kernel Transaction Undo Relog CHanGe - KTURLGU
    Layer 6 : Control File -  KCOCODCF     [tbs.h]
     
    Layer 10 : INDEX -  KCOCODIX     [kdi.h]
         Opcode 1 : load index block (Loader with direct mode) 
         Opcode 2 : Insert leaf row 
         Opcode 3 : Purge leaf row 
         Opcode 4 : Mark leaf row deleted 
         Opcode 5 : Restore leaf row (clear leaf delete flags) 
         Opcode 6 : Lock index block 
         Opcode 7 : Unlock index block 
         Opcode 8 : Initialize new leaf block 
         Opcode 9 : Apply Itl Redo 
         Opcode 10 :Set leaf block next link 
         Opcode 11 :Set leaf block previous link 
         Opcode 12 :Init root block after split 
         Opcode 13 :Make leaf block empty 
         Opcode 14 :Restore block before image 
         Opcode 15 :Branch block row insert 
         Opcode 16 :Branch block row purge 
         Opcode 17 :Initialize new branch block 
         Opcode 18 :Update keydata in row 
         Opcode 19 :Clear row's split flag 
         Opcode 20 :Set row's split flag 
         Opcode 21 :General undo above the cache (undo) 
         Opcode 22 :Undo operation on leaf key above the cache (undo) 
         Opcode 23 :Restore block to b-tree 
         Opcode 24 :Shrink ITL (transaction entries) 
         Opcode 25 :Format root block redo 
         Opcode 26 :Undo of format root block (undo) 
         Opcode 27 :Redo for undo of format root block 
         Opcode 28 :Undo for migrating block
         Opcode 29 :Redo for migrating block
         Opcode 30 :IOT leaf block nonkey update
         Opcode 31 :Cirect load root redo
         Opcode 32 :Combine operation for insert and restore rows 
     
    Layer 11 : Row Access -  KCOCODRW     [kdocts.h]
         Opcode 1 : Interpret Undo Record (Undo) 
         Opcode 2 : Insert Row Piece 
         Opcode 3 : Drop Row Piece 
         Opcode 4 : Lock Row Piece 
         Opcode 5 : Update Row Piece 
         Opcode 6 : Overwrite Row Piece 
         Opcode 7 : Manipulate First Column (add or delete the 1rst column) 
         Opcode 8 : Change Forwarding address 
         Opcode 9 : Change the Cluster Key Index 
         Opcode 10 :Set Key Links (change the forward & backward key links 
                    on a cluster key) 
         Opcode 11 :Quick Multi-Insert (ex: insert as select ...) 
         Opcode 12 :Quick Multi-Delete 
         Opcode 13 :Toggle Block Header flags 
     
    Layer 12 : Cluster -  KCOCODCL     [?]
    
    Layer 13 : Transaction Segment -  KCOCOTSG     [ktscts.h]
         Opcode 1 : Data segment format 
         Opcode 2 : Merge 
         Opcode 3 : Set link in block 
         Opcode 4 : Not used 
         Opcode 5 : New block (affects segment header) 
         Opcode 6 : Format block (affects data block) 
         Opcode 7 : Record link 
         Opcode 8 : Undo free list (undo) 
         Opcode 9 : Redo free list head (called as part of undo) 
         Opcode 9 : Format free list block (freelist group) 
         Opcode 11 :Format new blocks in free list 
         Opcode 12 :free list clear 
         Opcode 13 :free list restore (back) (undo of opcode 12) 
     
    Layer 14 : Transaction Extent -  KCOCOTEX     [kte.h]
         Opcode 1 : Add extent to segment 
         Opcode 2 : Unlock Segment Header 
         Opcode 3 : Extent DEaLlocation (DEL) 
         Opcode 4 : Undo to Add extent operation (see opcode 1) 
         Opcode 5 : Extent Incarnation number increment 
         Opcode 6 : Lock segment Header 
         Opcode 7 : Undo to rollback extent deallocation (see opcode 3) 
         Opcode 8 : Apply Position Update (truncate) 
         Opcode 9 : Link blocks to Freelist 
         Opcode 10 :Unlink blocks from Freelist 
         Opcode 11 :Undo to Apply Position Update (see opcode 8) 
         Opcode 12 :Convert segment header to 6.2.x type 
     
    Layer 15 : Table Space -  KCOCOTTS     [ktt.h]
        Opcode 1 : Format deferred rollback segment header 
        Opcode 2 : Add deferred rollback record 
        Opcode 3 : Move to next block 
        Opcode 4 : Point to next deferred rollback record 
     
    Layer 16 : Row Cache -  KCOCOQRC     
     
    Layer 17 : Recovery (REDO) -  KCOCORCV     [kcv.h]
         Opcode 1 : End Hot Backup : This operation clears the hot backup 
                    in-progress flags in the indicated list of files 
         Opcode 2 : Enable Thread : This operation creates a redo record 
                    signalling that a thread has been enabled 
         Opcode 3 : Crash Recovery Marker 
         Opcode 4 : Resizeable datafiles
         Opcode 5 : Tablespace ONline
         Opcode 6 : Tablespace OFFline
         Opcode 7 : Tablespace ReaD Write
         Opcode 8 : Tablespace ReaD Only
         Opcode 9 : ADDing datafiles to database
         Opcode 10 : Tablespace DRoP
         Opcode 11 : Tablespace PitR     
    Layer 18 : Hot Backup Log Blocks -  KCOCOHLB     [kcb.h]
         Opcode 1 : Log block image 
         Opcode 2 : Recovery testing 
     
    Layer 19 : Direct Loader Log Blocks - KCOCODLB     [kcbl.h]
         Opcode 1 : Direct block logging 
         Opcode 2 : Invalidate range 
         Opcode 3 : Direct block relogging
         Opcode 4 : Invalidate range relogging     
    Layer 20 : Compatibility Segment operations - KCOCOKCK  [kck.h]
         Opcode 1 : Format compatibility segment -  KCKFCS
         Opcode 2 : Update compatibility segment - KCKUCS
    Layer 21 : LOB segment operations - KCOCOLFS     [kdl2.h]
         Opcode 1 : Write data into ILOB data block - KDLOPWRI
    Layer 22 : Tablespace bitmapped file operations -  KCOCOTBF [ktfb.h]
 				 Opcode 1 : format space header - KTFBHFO
 	    	 Opcode 2 : space header generic redo - KTFBHREDO
 	  		 Opcode 3 : space header undo - KTFBHUNDO
 	   		 Opcode 4 : space bitmap block format - KTFBBFO
 				 Opcode 5 : bitmap block generic redo - KTFBBREDO 
    Layer 23 : write behind logging of blocks - KCOCOLWR [kcbb.h]
 			   Opcode 1 : Dummy block written callback - KCBBLWR
    Layer 24 : Logminer related (DDL or OBJV# redo) - KCOCOKRV [krv.h]
 				 Opcode : common portion of the ddl - KRVDDL
 	   		 Opcode : direct load redo - KRVDLR 
 	   	   Opcode : lob related info - KRVLOB
 	       Opcode : misc info - KRVMISC 
 		     Opcode : user info - KRVUSER
```

## 重做记录(Redo Record)

REDO 记录是由一组 CV 组成的，这组 CV 完成对数据库的一个原子修改操作。比如，一个 REDO 记录里可能包含 3 个 CV，第一个是对 UNDO SEGMENT HEADER 的修改，第二个是对 UNDO SEGMENT 的修改，第三个是对 DATA BLOCK 的修改。而一个事务可能包含多个 REDO 记录。

Record 头的长度是根据 VLD 的值得出的，已知的值有：

| Mnemonic | Value | Description                                                  |                                                              |
| -------- | ----- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| KCRVOID  | 0     | The contents are not valid.                                  | Record 内容无效                                              |
| KCRVALID | 1     | Includes change vectors                                      | Record 包含 change vectors                                   |
| KCRDEPND | 2     | Includes commit SCN                                          | Record 包含已提交的 SCN                                      |
| KCRVOID  | 4     | Includes dependent SCN                                       | 包含依赖 SCN，内部还有其他 SCN                               |
| KCRNMARK | 8     | New SCN mark record. SCN allocated exactly at this point in the redo log by this instance | Record 新的 SCN，此时，SCN 已在此实例的重做日志中精确分配    |
| KCRMARK  | 16    | Old SCN mark record. SCN allocated at or before this point in the redo. May be allocated by another instance | Record 旧的 SCN，重做日志中，在此点或之前分配的 SCN。可能由另一个实例分配 |
| KCRORDER | 32    | New SCN was allocated to ensure redo for some block would be ordered by inc/seq# when redo sorted by SCN | 分配了新的 SCN，确保在按 SCN 排序时，某些块的重做由 inc/seq# 排序 |

Record 头长度有两种情况：24字节 和 68字节，通常情况下是 24字节，在某些情况下会增加到 68字节。

目前，没有找到明确的资料说明哪些是某些情况。能搜索到的是：只要 VLD 的值中包括了4，长度就是68字节，反之是24字节。包括 4的意思是，比如 VLD: 0x0d，十进制就是 13=8+4+1，长度就是 68；VLD: 0x09，十进制就是 9=8+1，长度就是 24。仔细分析下这些值的组成，这个逻辑很容易使用位运算来处理！ 

## 一些案例

```c
REDO RECORD - Thread:1 RBA: 0x0000a1.000040ce.0010 LEN: 0x0054 VLD: 0x01 11
SCN: 0x0000.00f7a9c9 SUBSCN: 1 03/12/2008 09:37:49
CHANGE #1 TYP:0 CLS:33 AFN:2 DBA:0x00800111 SCN:0x000.00f7a9c7 SEQ: 1 OP:5.4
/* 
REDO 记录的 RBA 是 0x0000a1.000040ce.0010，转换成十进制就是 161.64.16， LOG SEQUENCE 号是 161，在 REDO 日志中的块号是 64，起始字节是块内的 16B。这个 REDO 记录的长度是 84B(0X54)。
“VLD:0X01”表示该 REDO 记录的类型，“0X01”表示 CHANGE VECTOR COPIED IN。 
“CHANGE #1”表示这是该 REDO 记录的第一个 CV。
根据上面的 OPCODE 清单可知，“OP:5.4” 表示 Commit transaction(transaction table update)，即修改事务表。
RDBA 是“2/273”，通过DBA_EXTENTS 查询为_SYSSMU3$。
*/ 
```

