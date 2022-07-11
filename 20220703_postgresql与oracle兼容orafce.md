PostgreSQL是和Oracle最接近的企业数据库，包括数据类型，功能，架构和语法等几个方面。甚至大多数的日常应用的性能也不会输给Oracle。

但是Oracle有些函数或者包，默认PostgreSQL是没有的，需要安装orafce包来实现这些兼容性。

例如现在orafce已经包含了如下内容。

- 类型 date, varchar2 and nvarchar2
- 函数 concat, nvl, nvl2, lnnvl, decode, bitand, nanvl, sinh, cosh, tanh and oracle.substr
- dual 表
- package :

```
        dbms_output  
        utl_file  
        dbms_pipe  
        dbms_alert  
        PLVdate  
        PLVstr and PLVchr  
        PLVsubst  
        DBMS_utility  
        PLVlex  
        DBMS_ASSERT  
        PLUnit  
        DBMS_random  
```

orafce的安装步骤如下：

http://pgxn.org/dist/orafce/

下载最新版本。

```
wget http://api.pgxn.org/dist/orafce/3.1.2/orafce-3.1.2.zip  
```

安装

```
unzip orafce-3.1.2.zip  
mv orafce-3.1.2 /opt/soft_bak/postgresql-9.4.5/contrib  
cd /opt/soft_bak/postgresql-9.4.5/contrib/orafce-3.1.2  
```

把pg_config命令放到当前路径，之后就可以编译安装。

```
export PATH=/opt/pgsql/bin:$PATH  
make clean  
make  
make install  
```

创建extension

```
su - postgres  
psql  
postgres=# create extension orafce;  
CREATE EXTENSION  
```

Oracle兼容 函数列表：

```
postgres=# \df  
```

Oracle兼容 dual表，在PG里用了一个视图来实现。

```
postgres=#  \dv  
        List of relations  
 Schema | Name | Type |  Owner     
--------+------+------+----------  
 public | dual | view | postgres  
(1 row)  
postgres=# \d+ dual  
                       View "public.dual"  
 Column |       Type        | Modifiers | Storage  | Description   
--------+-------------------+-----------+----------+-------------  
 dummy  | character varying |           | extended |   
View definition:  
 SELECT 'X'::character varying AS dummy;  
```

Oracle兼容 包列表：

在PostgreSQL里用schema+函数来实现。

```
postgres=# \dn  
     List of schemas  
     Name     |  Owner     
--------------+----------  
 dbms_alert   | postgres  
 dbms_assert  | postgres  
 dbms_output  | postgres  
 dbms_pipe    | postgres  
 dbms_random  | postgres  
 dbms_utility | postgres  
 madlib       | postgres  
 oracle       | postgres  
 plunit       | postgres  
 plvchr       | postgres  
 plvdate      | postgres  
 plvlex       | postgres  
 plvstr       | postgres  
 plvsubst     | postgres  
 public       | postgres  
 utl_file     | postgres  
(16 rows)  
```

例如dbms_alert包:

```
postgres=# \df dbms_alert.*  
                                                            List of functions  
   Schema   |      Name      | Result data type |                              Argument data types                              |  Type     
------------+----------------+------------------+-------------------------------------------------------------------------------+---------  
 dbms_alert | _signal        | void             | name text, message text                                                       | normal  
 dbms_alert | defered_signal | trigger          |                                                                               | trigger  
 dbms_alert | register       | void             | name text                                                                     | normal  
 dbms_alert | remove         | void             | name text                                                                     | normal  
 dbms_alert | removeall      | void             |                                                                               | normal  
 dbms_alert | set_defaults   | void             | sensitivity double precision                                                  | normal  
 dbms_alert | signal         | void             | _event text, _message text                                                    | normal  
 dbms_alert | waitany        | record           | OUT name text, OUT message text, OUT status integer, timeout double precision | normal  
 dbms_alert | waitone        | record           | name text, OUT message text, OUT status integer, timeout double precision     | normal  
(9 rows)  
```

#### 