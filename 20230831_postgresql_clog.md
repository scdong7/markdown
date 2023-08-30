# 1 Commit Log(clog)

clog用来记录事务号和事务号的状态，主要是用来判断事务的可见性。每个事务状态占用两个bit位。事务的状态有4个：IN PROGRESS，COMMITTED，ABORTED和SUB_COMMITTED。clog文件大小8K页，存放一个数组，数组的每个元素对应事务ID号和事务状态。

PG的clog存放在缓存中，当checkpoint时开始刷新到clog文件中。当PG库down库后，clog会被写入到$PGDATA/pg_xact子目录中，文件命名为0000,0001,00002......，单个文件最大为256K。当PG库启动时，这些文件会被加载到内存中。clog的信息同样会被记录到wal日志中，当数据库异常中断时，clog的信息会从wal日志还原。

PG的clog定义可以从文件/src/include/access/clog.h中获取。

```c
#define TRANSACTION_STATUS_IN_PROGRESS		0x00
#define TRANSACTION_STATUS_COMMITTED		0x01
#define TRANSACTION_STATUS_ABORTED			0x02
#define TRANSACTION_STATUS_SUB_COMMITTED	0x03
```

PG的clog文件存放内容信息可以从文件/src/backend/access/transam/clog.c中获取

```c
/* We need two bits per xact, so four xacts fit in a byte */
#define CLOG_BITS_PER_XACT	2
#define CLOG_XACTS_PER_BYTE 4
#define CLOG_XACTS_PER_PAGE (BLCKSZ * CLOG_XACTS_PER_BYTE)
#define CLOG_XACT_BITMASK	((1 << CLOG_BITS_PER_XACT) - 1)

#define TransactionIdToPage(xid)	((xid) / (TransactionId) CLOG_XACTS_PER_PAGE)
#define TransactionIdToPgIndex(xid) ((xid) % (TransactionId) CLOG_XACTS_PER_PAGE)
#define TransactionIdToByte(xid)	(TransactionIdToPgIndex(xid) / CLOG_XACTS_PER_BYTE)
#define TransactionIdToBIndex(xid)	((xid) % (TransactionId) CLOG_XACTS_PER_BYTE)
```

# 2 clog原理

pg中就是通过clog来存储事务的xid和事务的状态。正因为如此，我们会发现在pg中如果想要取消一个执行了很长时间的事务，基本上是瞬间完成的，而不是像Oracle中一样需要等到undo表空间中内容回滚完，因为pg中只需要将事务的状态由IN_PROGRESS修改为ABORTED即可。但这样同时也会导致一个问题，如果执行了上述的操作，那么表中必然会存在大量ABORTED状态的数据，当我们后面对该表进行查询的时候，还会去对这些数据进行扫描判断其clog的事务状态，性能必然会很低，那pg中是如何解决这类问题的呢？

对于pg的32位xid而言，按照xid最大20亿来估算，clog最大也就：20亿/4/1024/1024 = 476MB，竟然还不到500MB，那么查询效率必然不会太低。

在pg中，事务id并不是在事务开始时就会被分配，而是当事务进行了数据修改之类的操作时才会分配xid，而当事务提交或回滚时，其事务状态便会被写入clog中。

正如我们所知，pg中update一行数据时，实质是对旧的数据打上删除的标记，然后再其旁边再插入一行新的数据，同时为了保证数据的可见性，必然也会对其记录一个事务的标记。

例如当我们在事务中写入数据时，可以看到这个事务的状态是IN_PROGRESS，而当我们提交或者回滚该事务时，如果再去修改这个状态必然对性能会产生一定的影响，而且这个操作也是多余的。为什么呢？正如我们前面所说，pg中判断事务的可见性时通过xid去clog中查找事务的状态的，既然查找该数据的时候会去判断并且将改行数据的状态进行修改，那么我们为什么还要在事务提交时进行修改呢？

正因为如此，pg中是这么做的：**前一个事务所有修改的数据，它没有在提交或者回滚的当时改掉所有的修改标记，而是等到下次查询时进行修改。**

既然我们修改了行的事务状态，那么该行的数据块的校验和就变了，那是不是应该要产生wal日志呢？即使没有产生数据的变化，不然像流复制这种通过wal日志同步的主备环境中，主库的数据块变化了如果不产生wal日志那备库的数据块不是没变吗？那么怎么保证数据可见性的一致呢？

但事实上并不是这样，只有当设置了checksum时才会在设置标记位的时候去写WAL日志。而且也并不是每次设置标记位的时候都去写，而是前一次checkpoint之后，数据块第一次被修改时才会写整个数据块到WAL日志中。

可以看到和Oracle一样，pg中简单的select操作也是可能会产生wal日志的！

# 3 clog 丢失

1. 如果正常关闭数据库，然后删除clog文件，启动会报错

   ```shell
   "could not access status of transaction 12726", "Could not open file ""pg_xact/0000"": No such file or directory.
   ```
   解决方法：可以初始化一个clog文件，然后启动数据库，由于事务的状态丢失，会导致某些事务状态丢失。

   ```shell
   dd if=/dev/zero of=$PGDATA/pg_xact/0000 bs=8k count=1
   ```
2. 如果异常down库，然后删除clog文件，启动PG会重新生成clog文件。
