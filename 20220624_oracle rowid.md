# 1 ROWID分类

- 物理rowid : [普通的堆表中的rowid是物理rowid]
  - 限制rowid(restricted rowid)
  - 扩展rowid(extended rowid)
- 逻辑rowid : [索引组织表(IOT)的rowid是逻辑rowid)]

**oracle提供了一种urowid的数据类型，同时支持物理和逻辑rowid。** 

**限制rowid是oracle7以前的rowid格式，现在已经不再使用，保留该类型是为了兼容性。本文的提到物理rowid一般指扩展rowid。**

## 1.1 Restricted ROWID Format

Restricted ROWID占用6 个字节存储空间(10bit file#+22bit block#+16bit row#) rowid格式为:BBBBBBBB.RRRR.FFFF。

## 1.2 Extended ROWID Format

Extended  ROWID占用10个字节存储空间(32bit object#+10bit rfile#+22bit block#+16bit row#) 

在内部，存储的10个字节(bytes)，即80位(bit)又按如下规则进行划分：

- 数据对象编号 32 bit
- 相关文件编号 10 bit
- 块编号             22 bit
- 行编号             16 bit

扩展的ROWID 使用以64 为基数的编码方案的18 个字符来显示，该方案将六个位置用于数据对象编号、三个位置用于相关文件编号、六个位置用于块编号、三个位置用于行编号。

文件号仍然用10位表示,只是不再需要置换,为了向后兼容,同时引入了相对文件号(rfile#),所以从Oracle7到8,Rowid仍然无需发生变化.

rdba(Tablespace relative database block address)就是rowid中的rfile#+block#.

rowid这样改变之后，数据库中数据库文件个数的限制从整个数据库最多只能有的2^10-2=1022个数据文件(去掉全0和全1), 变为了每个表空间中可以最多有2^10-2个数据文件。所以说，数据库能支持的数据文件最大数是受rowid的长度限制的。那么增加的32bit object# 这个前缀主要就是用来定位表空间的，同时这个object#其实对应的就是data_object_id，由于一个段对象只能属于一个表空间，同时data_object_id就是标识了一个段的物理存储id.因此object#+rfile#就可以唯一定位当前的rowid是在那个数据文件上了。

# 2 dbms_rowid包

```sql
select rowid,
 dbms_rowid.rowid_object(rowid) object_id,-- 数据对象号
 dbms_rowid.rowid_relative_fno(rowid) file_id, --相对文件号
 dbms_rowid.rowid_block_number(rowid) block_id, --在第几个块
 dbms_rowid.rowid_row_number(rowid) num  --在block中的行数
from test ;

rowid是伪列（pseudocolumn）实际上这一列本身在数据字典中并不存在，在查询结果输出时它被构造出来的。
rowid并不会真正存在于表的data block中,但是他会存在于index当中，用来通过rowid来寻找表中的行数据。
```
