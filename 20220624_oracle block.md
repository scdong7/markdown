# block dump

    转储一个block:alter system dump datafile 4 block 20;
    转储多个block:alter system dump datafile <file_id> block min <block_id> block max <block_id+blocks-1>;
    
    一共占据48bytes，包括24bytes的控制信息，和一系列的Interested Transaction Slot (ITS)。这些ITS组合在一起称为Interested Transaction List (ITL)。初始的ITL slot 数量由 INITRANS 决定（index branch block 只有1个slot）。如果有足够的剩余空间，oracle会根据需要动态的分配这些slot，直到受到空间限制或者达到了MAXTRANS。
         
    Block header dump:  0x01818704
     Object id on Block? Y
     seg/obj: 0x12e8e  csc: 0x959.d759d143  itc: 2  flg: E  typ: 1 - DATA
         brn: 0  bdba: 0x1818701 ver: 0x01 opc: 0
         inc: 0  exflg: 0
    
    Itl           Xid           Uba                  Flag      Lck        Scn/Fsc
    0x01   0x0001.028.0000092d   0x0080438c.0226.16   C---      0          scn	0x0959.d759d18a
    0x02   0x0008.01c.00000abf   0x00808842.029c.2d   ----      22        fsc 0x0000.00000000 
    
    24bytes的控制信息包括：
    
    Object number（seg/obj）: 占用4bytes，指在OBJ$中记录的segment 的 object number(0x806d=32877)
    
    Cleanout SCN（csc）:  占用6bytes，最后一次 full cleanout 的scn
    
    ITL count（itc）: 占用1byte，ITL 的slot数量。
    
    Flag: 占用2bytes。O表示这个block在freelist 上。否则flag为”-”
    
    Block type: 占用1byte。1=data; 2=index
    
    ITL freelist slot（fsl）: 占用1byte。Index to the first slot on the ITL freelist
    
    Next freelist block（fnx）: 占用4byte。Segment freelist中下一个block的RDBA
    
    Version: 1 byte
    
    Unused: 4bytes，用来前后兼容。
    
    每个ITL entry包括以下的内容：
    
    Transaction id（Xid）: 8bytes。其中包括rollback segment number, transaction table中的slot number等。
    
    Undo block address（Uba）: 8bytes。其中包括rollback segment block的DBA，sequence number等。
    
    Flags: 1nibble。---- = transaction is active, or committed pending cleanout
    C--- = transaction has been committed and locks cleaned out
    -B-- = this undo record contains the undo for this ITL entry
    --U- = transaction committed (maybe long ago); SCN is an upper bound
    ---T = transaction was still active at block cleanout SCN
    
    Locks: 3nibbles. 也就是所谓的行级锁（row-level locks）
    
    SCN or free space credit: 6bytes. 如果这个事务已经clean out，这个值就是SCN；否则，前两个字节表示由这个事务释放的此block中的空间数。
    
    uba(undo block address)
    
    DBA（Data Block Address）一般指绝对数据块地址. rowid用来表示一行的物理地址，一行唯一确定一个rowid，并且在使用中一般不会改变，除非rowid之后在行的物理位置发生改变的情况下才会发生变化。在rowid 中，就有一段是来表示DBA的。
    
    Rdba（Tablespace relative database block address）是相对数据块地址，dba一般指绝对数据块地址。如果数据文件超过>=1024即2的10次方的话,仅用绝对dba在内部没办法表示的，内部一般是用rdba来表示,它限定了某个表空间。rowid的格式也是用10位表示的数据文件,只能说是ORACLE当初设计时导致的。

# block中的uba,scn的作用

    当我们想要查找某个数据的时候，发现这个数据块的版本比我们要查找的要新，那么我们只能从UNDO中去查找这个数据的前映像（PRE IMAGE），在回滚段中找到这个数据的前映像后，把前映像和CURRENT的数据块合并，旧形成一个CR BLOCK，这样，通过查询CR BLOCK就可以得到一致性的数据了。这个CR BLOCK的生成不是在PGA里进行的，而是要在DB CACHE里申请一个数据块，然后生成CR BLOCK。这个CR BLOCK创建后，如果下一回还要访问这个版本的数据块，那么就不需要创建新的CR BLOCK，而直接可以使用以前创建的CR BLOCK了。每创建一个CR BLOCK，CR BLOCKS CREATED计数器就会增加1。
    
    由于CR BLOCK和CURRENT BLOCK的RDBA都是相同的，因此它们会被放到相同的HASH链上。因此如果某些BLOCK的CR BLOCK的版本过多，也会导致BUFFER CACHE CHAINS闩锁竞争加剧。
    如果在生成CR BLOCK的过程中，发现UNDO中的PRE IMAGE由于提交的时间比较长，已经被覆盖，那么，就会出现著名的ORA-1555。
    如果单位时间内CR BLOCKS CREATED较大，那么应该检查系统的CPU资源是否出现瓶颈，另外要注意检查BUFFER CACHE相关的闩锁，是否存在竞争。
