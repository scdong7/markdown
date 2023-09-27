# 1 流复制

流复制出现是从2010年推出的pg9.0开始的，其历史大致为：

1. **起源** ：pg9.0开始支持流式物理复制，用户可以通过流式复制，构建只读备库(主备物理复制，块级别一致)。流式物理复制可以做到极低的延迟(通常在1毫秒以内)。
2. **同步流复制** ：pg9.1开始支持同步复制，但是当时只支持一个同步流复制备节点(例如配置了3个备，只有一个是同步模式的，其他都是异步模式)。同步流复制的出现，保证了数据的0丢失。
3. **级联流复制** ：pg9.2支持级联流复制。即备库还可以再连备库。
4. **流式虚拟备库** ：pg9.2还支持虚拟备库，即就是只有WAL，没有数据文件的备库。
5. **逻辑复制** ：pg9.4开始可以实现逻辑复制，逻辑复制可以做到对主库的部分复制，例如表级复制，而不是整个集群的块级一致复制。
6. **增加多种同步级别** ：pg9.6版本开始可以通过synchronous_commit参数，来配置事务的同步级别

# 2 物理复制

PostgreSQL物理流复制大致过程为：

1. PG软件安装
2. postgresql.conf参数配置
3. pg_hba.conf配置
4. pg_basebackup方式部署备库
5. 配置简单
6. 备库与主库物理完全一致，并支持只读

## 2.1 异步流复制Hot StandBy实例

| IP            | 角色    | 数据目录            |
| ------------- | ------- | ------------------- |
| 192.168.56.31 | primary | /opt/pgsql14.2/data |
| 192.168.56.32 | standby | /opt/pgsql14.2/data |

主库主要配置参数

```c
wal_level = replica
max_wal_senders = 10 
```

修改pg_hba.conf允许主库接受流复制连接

```c
/*CREATE ROLE replica login replication encrypted password '123456'*/
host replication replica 192.168.56.0/24 trust
```

standby生成基础备份

```c
pg_basebackup -h 192.168.56.31 -U replica -F p -R -D /opt/pgsql14.2/data -l replbackup20220516
```

standby修改配置参数

```c
hot_standby = on
```

## 2.2 同步流复制Hot StandBy实例

| IP                                                                    | 角色      | 数据目录            |
| --------------------------------------------------------------------- | --------- | ------------------- |
| 192.168.56.31                                                         | primary   | /opt/pgsql14.2/data |
| 192.168.56.32                                                         | standby01 | /opt/pgsql14.2/data |
| 192.168.56.33                                                         | standby02 | /opt/pgsql14.2/data |
| 同步最好有多个，如果只有一个standby，standby出现问题，primary会hang住 |           |                     |

主库主要配置参数

```c
wal_level = replica
max_wal_senders = 10 
synchronous_standby_names='*'
```

修改pg_hba.conf允许主库接受流复制连接

```c
/*CREATE ROLE replica login replication encrypted password '123456'*/
host replication replica 192.168.56.0/24 trust
```

standby01/02生成基础备份

```c
pg_basebackup -h 192.168.56.31 -U replica -F p -R -D /opt/pgsql14.2/data -l replbackup20220516
```

standby01/02修改配置参数

```c
hot_standby = on
primary_conninfo='application_name=standby01/02 user=replica password=123456 host=192.168.56.31 port=5432 options=''-c wal_sender_timeout=5000'''
```

## 2.3 数据字典pg_stat_replication

通过查询数据字典，验证测试。

## 2.4 复制槽slot

因为pg在归档模式下，对于已经完成归档的wal日志会自动清理，所以提供了复制槽来避免主库在所有的备库收到 WAL 日志之前不会移除它们，并且主库也不会移除可能导致恢复冲突的行，即使备库断开也是如此。

**例子：**

创建一个复制槽：

```sql
bill=# SELECT * FROM pg_create_physical_replication_slot('node_a_slot'); 
slot_name | lsn
-------------+-----
 node_a_slot |
bill=# SELECT slot_name, slot_type, active FROM pg_replication_slots; slot_name | slot_type | active
-------------+-----------+-------- 
node_a_slot | physical | f
(1 row)
```

要配置后备机使用这个槽，在备库的recovery.conf中应该配置 primary_slot_name，例如：

```shell
standby_mode = 'on'
primary_conninfo = 'host=192.168.7.180 port=1921 user=bill password='xxx' primary_slot_name = 'node_a_slot'
```
