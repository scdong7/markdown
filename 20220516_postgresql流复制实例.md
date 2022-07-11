# 流复制

## 异步流复制Hot StandBy实例

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
ot_standby = on
```

## 同步流复制Hot StandBy实例

| IP                                                           | 角色      | 数据目录            |
| ------------------------------------------------------------ | --------- | ------------------- |
| 192.168.56.31                                                | primary   | /opt/pgsql14.2/data |
| 192.168.56.32                                                | standby01 | /opt/pgsql14.2/data |
| 192.168.56.33                                                | standby02 | /opt/pgsql14.2/data |
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

## 数据字典pg_stat_replication

通过查询数据字典，验证测试。