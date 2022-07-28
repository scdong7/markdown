## 关系和元组的组织方式

- 普通堆(ordinary cataloged heap)

- 临时堆(temporary heap)

- 序列(sequence relation)

- TOAST表

  每个堆文件都是由多个文件块组成的。

### BLOCK结构

```c
 * +----------------+---------------------------------+
 * | PageHeaderData | linp1 linp2 linp3 ...           |
 * +-----------+----+---------------------------------+
 * | ... linpN |									 									  |
 * +-----------+--------------------------------------+
 * |		   ^ pd_lower																  |
 * |												                          |
 * |			 v pd_upper							                    |
 * +-------------+------------------------------------+
 * |			 			 | tupleN ...                         |
 * +-------------+------------------+-----------------+
 * |	   ... tuple3 tuple2 tuple1   | "special space" |
 * +--------------------------------+-----------------+
 *									                ^ pd_special 
   
 typedef struct PageHeaderData
{
	/* XXX LSN is member of *any* block, not only page-organized ones */
	PageXLogRecPtr pd_lsn;		/* LSN: next byte after last byte of xlog
								 						 * record for last change to this page */
	uint16		pd_checksum;	/* checksum */
	uint16		pd_flags;		/* flag bits, see below */
	LocationIndex pd_lower;		/* offset to start of free space */
	LocationIndex pd_upper;		/* offset to end of free space */
	LocationIndex pd_special;	/* offset to start of special space */
	uint16		pd_pagesize_version;
	TransactionId pd_prune_xid; /* oldest prunable XID, or zero if none */
	ItemIdData	pd_linp[FLEXIBLE_ARRAY_MEMBER]; /* line pointer array */
} PageHeaderData;

typedef struct ItemIdData
{
	unsigned	lp_off:15,		/* offset to tuple (from start of page) */
				    lp_flags:2,		/* state of line pointer, see below 元组状态 未使用、正常使用、HOT重定向、死亡*/
				    lp_len:15;		/* byte length of tuple */
} ItemIdData;
```

### HeapTuple元组

```c
typedef struct HeapTupleFields
{
	TransactionId t_xmin;		/* inserting xact ID */
	TransactionId t_xmax;		/* deleting or locking xact ID */

	union
	{
		CommandId	t_cid;		/* inserting or deleting command ID, or both */
		TransactionId t_xvac;	/* old-style VACUUM FULL xact ID */
	}			t_field3;
} HeapTupleFields;

typedef struct DatumTupleFields
{
	int32		datum_len_;		/* varlena header (do not touch directly!) */

	int32		datum_typmod;	/* -1, or identifier of a record type */

	Oid			datum_typeid;	/* composite type OID, or RECORDOID */

	/*
	 * datum_typeid cannot be a domain over composite, only plain composite,
	 * even if the datum is meant as a value of a domain-over-composite type.
	 * This is in line with the general principle that CoerceToDomain does not
	 * change the physical representation of the base type value.
	 *
	 * Note: field ordering is chosen with thought that Oid might someday
	 * widen to 64 bits.
	 */
} DatumTupleFields;
typedef struct ItemPointerData
{
	BlockIdData ip_blkid;
	OffsetNumber ip_posid;
}
struct HeapTupleHeaderData
{
	union
	{
		HeapTupleFields t_heap;/*记录对元组执行插入/删除操作的事物ID和命令ID，主要用于并发控制时检查元组对事务的可见性*/
		DatumTupleFields t_datum;/*当一个新元组在内存中形成的时候，并不关心事务的可见性，因此在t_choice中只需用DatumTupleFields
															机构来记录元组的长度等信息。但在把该元组插入到表文件时，需要在元组头信息中记录插入该元组的事物ID和															命令ID，故此时会把t_choice所占用的内存转换为HeapTupleFields结构并填充相应数据后在进行元组的插            															 入。
														 */
	}	t_choice;

	ItemPointerData t_ctid;		/* current TID of this or newer tuple  记录当前元组或新元组的物理位置
	                            (or a speculative insertion token) */
       /* Fields below here must match MinimalTupleData! */

  #define FIELDNO_HEAPTUPLEHEADERDATA_INFOMASK2 2
	uint16		t_infomask2;	/* number of attributes + various flags */
  #define FIELDNO_HEAPTUPLEHEADERDATA_INFOMASK 3
	uint16		t_infomask;		/* various flag bits, see below */
  #define FIELDNO_HEAPTUPLEHEADERDATA_HOFF 4
	uint8		t_hoff;			/* sizeof header incl. bitmap, padding */
  /* ^ - 23 bytes - ^ */
  #define FIELDNO_HEAPTUPLEHEADERDATA_BITS 5
	bits8		t_bits[FLEXIBLE_ARRAY_MEMBER];	/* bitmap of NULLs */
  /* MORE DATA FOLLOWS AT END OF STRUCT */
};
typedef HeapTupleHeaderData *HeapTupleHeader;
typedef struct HeapTupleData
{
	uint32		t_len;			/* length of *t_data */
	ItemPointerData t_self;		/* SelfItemPointer */
	Oid			t_tableOid;		/* table the tuple came from */
#define FIELDNO_HEAPTUPLEDATA_DATA 3
	HeapTupleHeader t_data;		/* -> tuple header and data */
} HeapTupleData;

typedef HeapTupleData *HeapTuple;

typedef struct TupleDescData
{
	int			natts;			/* number of attributes in the tuple */
	Oid			tdtypeid;		/* composite type ID for tuple type */
	int32		tdtypmod;		/* typmod for tuple type */
	int			tdrefcount;		/* reference count, or -1 if not counting */
	TupleConstr *constr;		/* constraints, or NULL if none */
	/* attrs[N] is the description of Attribute Number N+1 */
	FormData_pg_attribute attrs[FLEXIBLE_ARRAY_MEMBER];
}			TupleDescData;
typedef struct TupleDescData *TupleDesc;
```

### HOT机制

HOT是“Heap Only Tuple”（仅元组堆）的缩写（与Overflow Update CHaining相比，它的缩写更好）。PG对于元组采用多版本技术存储，即使没有更新索引属性，也会在索引中产生新版本。HOT此功能克服了PostgreSQL处理UPDATE 效率低下的问题。

```c
当更新满足如下条件时称为HOT元组
1.所有索引属性都没有被修改（索引键是逐行判读的，若update修改了某属性，但前后值相同则认为没有修改）
2.更新的元组新旧版本在同一文件块内（限制同一文件块目的是为了通过版本链向后找时不产生额外的I/O操作而影响性能）
  HOT元组会被打上HEAP_ONLY_TUPLE标，HOT元组的上一个版本则被打上HEAP_HOT_UPDATE标志。更新一条HOT元组将不会在索引中引入新版本，
  当通过索引获取元组时首先会找到同一块中最老的版本，然后顺着版本链向后找，直到遇到HOT元组为止。
  HOT消除了拥有完全相同键值的索引记录，减少了索引大小
```

## pageinspect插件

```sql
SELECT * FROM page_header(get_raw_page('tb2', 0));
SELECT * FROM heap_page_items(get_raw_page('tb2', 0));
SELECT * FROM heap_page_item_attrs(get_raw_page('tb2', 0), 'tb2'::regclass);
SELECT tuple_data_split('tb2'::regclass, t_data, t_infomask, t_infomask2, t_bits) FROM heap_page_items(get_raw_page('tb2', 0));
```



## 表操作

### 打开表

​	返回RelationData结构，并不是打开具体的物理文件。

- ​    OID打开

  ​	调用Relation relation_open(Oid relationId, LOCKMODE lockmode)

- ​    表名打开

   	调用Relation relation_openrv(const RangeVar *relation, LOCKMODE lockmode)

### 顺序扫描表

#### 基本与同步扫描策略

思路：a,b,c三个对t表的扫描，a最快b其次c最慢。都从0块开始扫描。理论上a扫描的文件块都应该缓存在缓冲池中，b,c 访问的都可以在缓冲池中命中，但实际上因为大量并发进程，缓冲池大小有限。a缓冲的缓冲块可能已经被替换出去。b,c 无法命中。同步扫描的思路就是，a 从0 块开始扫描，b,c 从a 扫描的当前位置扫描。这样a被换出的可能性就极小，可以有效降低IO开销。
由于b,c不是从第0块开始，所以b,c扫描到N块时，需要回转到第0块，将第0块到起始位置之间的文件块继续扫完。这是虽然b,c不能在共享a的缓冲区数据，但b,c之间仍然能攻共享。

PG通过在共享内存放置ss_scan_locations_t结构来存放对于某个表的当前扫描位置，每个进程都有一个全局指针指向ss_scan_locations_t结构。目前PG中只有在建立B-tree,Hash和GisT三种索引时才可能用到同步扫描。（需判断需要扫描的文件块总数是否超过缓冲区总数的1/4 重设rs_allow_sync和rs_syncscan）

```c
typedef struct ss_scan_location_t
{
	RelFileNode relfilenode;	/* identity of a relation */
	BlockNumber location;		/* last-reported location in the relation */
} ss_scan_location_t;

typedef struct ss_lru_item_t
{
	struct ss_lru_item_t *prev;
	struct ss_lru_item_t *next;
	ss_scan_location_t location;
} ss_lru_item_t;
typedef struct ss_scan_locations_t
{
	ss_lru_item_t *head;
	ss_lru_item_t *tail;
	ss_lru_item_t items[FLEXIBLE_ARRAY_MEMBER]; /* SYNC_SCAN_NELEM items */
} ss_scan_locations_t;
```

#### 扫描描述符HeapScanDescData

```c
typedef struct TableScanDescData
{
	/* scan parameters */
	Relation	rs_rd;			/* heap relation descriptor */
	struct SnapshotData *rs_snapshot;	/* snapshot to see */
	int			rs_nkeys;		/* number of scan keys */
	struct ScanKeyData *rs_key; /* array of scan key descriptors */
	/* Range of ItemPointers for table_scan_getnextslot_tidrange() to scan. */
	ItemPointerData rs_mintid;
	ItemPointerData rs_maxtid;
	/*
	 * Information about type and behaviour of the scan, a bitmask of members
	 * of the ScanOptions enum (see tableam.h).
	 */
	uint32		rs_flags;
	struct ParallelTableScanDescData *rs_parallel;	/* parallel scan
													 * information */
} TableScanDescData;
typedef struct TableScanDescData *TableScanDesc;
typedef struct HeapScanDescData
{
	TableScanDescData rs_base;	/* AM independent part of the descriptor */
															/* state set up at initscan time */
	BlockNumber rs_nblocks;			/* total number of blocks in rel */
	BlockNumber rs_startblock;	/* block # to start at */
	BlockNumber rs_numblocks;		/* max number of blocks to scan */
															/* rs_numblocks is usually InvalidBlockNumber, meaning "scan whole rel" */
															/* scan current state */
	bool		rs_inited;					/* false = scan not init'd yet */
	BlockNumber rs_cblock;			/* current block # in scan, if any */
	Buffer		rs_cbuf;					/* current buffer in scan, if any */
															/* NB: if rs_cbuf is not InvalidBuffer, we hold a pin on that buffer */
															/* rs_numblocks is usually InvalidBlockNumber, meaning "scan whole rel" */
	BufferAccessStrategy rs_strategy;	/* access strategy for reads */
	HeapTupleData rs_ctup;		/* current tuple in scan, if any */
	/*
	 * For parallel scans to store page allocation data.  NULL when not
	 * performing a parallel scan.
	 */
	ParallelBlockTableScanWorkerData *rs_parallelworkerdata;
	/* these fields only used in page-at-a-time mode and for bitmap scans */
	int			rs_cindex;		/* current tuple's index in vistuples */
	int			rs_ntuples;		/* number of visible tuples on page */
	OffsetNumber rs_vistuples[MaxHeapTuplesPerPage];	/* their offsets */
}			HeapScanDescData;
typedef struct HeapScanDescData *HeapScanDesc;
```

#### 扫描调用步骤

 heap_beginscan-->initscan-->heap_getnext-->heap_endscan

### 索引扫描表(略)

### 创建表

​	创建之前先判读目录是否存在，不存在则创建相应目录。然后调用mdcreate函数立即申请创建文件。

### 关闭表

   关闭表就是一个内存释放和将脏数据写回磁盘的过程。

### 删除表

  释放物理内存，删除表文件。PG将要删除的表加入到PendingRelDelete链表中，SMGR管理器调用smgrDoPendingDeletes函数将表删除。

```c
typedef struct PendingRelDelete
{
	RelFileNode relnode;		/* relation that may need to be deleted */
	BackendId	backend;		/* InvalidBackendId if not a temp rel */
	bool		atCommit;		/* T=delete at commit; F=delete at abort */
	int			nestLevel;		/* xact nesting level of request */
	struct PendingRelDelete *next;	/* linked-list link */
} PendingRelDelete;
/*注意:在新创建一个表时,该表也会被记录到PendingRelDelete链表,atCommit为true表示事物提交则删除,false新创建的表,事务失败就删除*/
```

## 元组操作

### 插入

调用HeapTuple heap_form_tuple(TupleDesc tupleDescriptor,Datum *values,bool *isnull)构建内存结构

调用void heap_insert(Relation relation, HeapTuple tup, CommandId cid,int options, BulkInsertState bistate)向表中插入元组。

### 删除

TM_Result heap_delete(Relation relation, ItemPointer tid,CommandId cid, Snapshot crosscheck, bool wait,TM_FailureData *tmfd, bool changingPart)，被删除的元组会通过VACUMM收回。

### 更新

TM_Result heap_update(Relation relation, ItemPointer otid, HeapTuple newtup,CommandId cid, Snapshot crosscheck, bool wait, TM_FailureData *tmfd, LockTupleMode *lockmode)
