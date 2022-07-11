# plproxy简介

plproxy 是一款能在PostgreSQL数据库中实现水平拆分的软件。项目主页：https://plproxy.github.io

- plproxy是skype开发的一个数据库组件
- plproxy本身并不是一个可以独立运行的进程，而只是安装到PG数据库中的一种PL语言（与PLPgsql一样的语言），但它可以把请求路由到后端的其他数据库中。路由到后端的请求，不是任意的sql请求，而是对一个函数的调用，所以plproxy实际上是把对一个函数的调用 路由到后端的数据库中。

- 它的理念是代理远程函数体内指定。函数调用同样标签创建的函数，所以，代理的目标信息需要在代理函数体内指定。 

# plproxy特性

- plproxy中的一个函数可以被一个或多个甚至全部后端节点执行。因为每一个查询是以auto-commit的方式在后端被执行的，所以 plproxy本质上是不支持事务的，使用时需要把一个原子操作封装到函数中提供给用户。
- plproxy后端数据库的节点数必须时2^n(n次方)，比如3台机器，第1台3个库，第2台3个库，第3台2个库。

# plproxy函数

plproxy语言实现的函数只需要定义路由功能，所以比较简单，目前支持如下几个语句。

- ```sql
  connect 'libpq connstr' |connect connect_fun(...) |argname; --指定连接到哪一个后端数据库中
  ```

-  ```sql
   cluster 'cluster_name' |connect cluster_fun(...) |argname;--定义函数调用会路由到哪一个集群中，实际发送到集群的那一台机器时由run on 语句决定。
   ```

-  ```sql
   run on all | any | <NR> |partition_func(...);--定义函数调用发送到具体哪一台机器
   all 表示在集群的每台机器上并行执行,然后把结果合并返回.
   any 表示随机挑选一台执行请求。
   <NR> 是一个后端数据库数据分片的数字，表示在那台后端数据库上运行。
   partition_func(...) 返回一个或多个hash值，然后将请求转到对应的后端数据库。
   ```

- ```sql
  split array_arg1 [,array_arg2 ...];
  split all;
  split语句必须于run on语句一起使用，并在run on之前。它可以使数组成为plproxy函数的参数，然后把数组中的元素分别发送到后端的多台数据库上并行执行。
  ```

- ```sql
  target other_function;
  target 可以让后端的函数名于plproxy的函数名不一样。
  ```

- ```sql
  select 默认情况下，PL/Proxy根据自己的签名生成查询。但这可以通过给出显式的SELECT语句来覆盖。
  SELECT之后的所有内容，直到分号被作为SQL传递。只对内容进行参数替换，否则将取消分析文本。为了避免将表列解析为函数参数，应使用表别名。查询结果的列数应与函数结果相同，名称也应相同。
  ```

```sql
例如：
create or replace function insert_user(i_username text,i_emailaddress text )returns integer 
as $$
cluster 'osdba_cluster01';
run on hashtext(i_username);
$$ language PL/Proxy;

/*
当用户调用plproxy的这个函数时，plproxy会根据函数内定义的路由语句'run on hashtext(i_username)' ,hash路由到后端实际数据库主机上执行同名函数。后端实际函数如下
*/
create or replace function insert_user(i_username text,i_emailaddress text )returns integer 
as $$
insert into osdba_user(username,email) values(i_username,i_emailaddress);
select 1;
$$ language sql;
```

# plproxy配置

plproxy即能配置成connect模式，也可以配置cluster模式。

- connect模式下，plproxy直接把请求路由到指定的后端数据库中。
- cluster模式下，plproxy支持数据水平分区，即可以把数据水平拆分到多台后端数据库中。需要事先配置集群。

cluster模式下集群配置方式有以下两种：

- 使用cluster configuration API方式
- 使用SQL/MED方式（*拥有更好的性能）

## cluster configuration API方式

主要定义3个函数。

- plproxy.get_cluster_version(cluster_name text)

  用户定义集群版本，它的作用时在线升级集群时，能改变集群的配置。此函数在处理请求前被调用，当返回的版本号比之前大是，plproxy需要调用另两个集群配置函数get_cluster_partitions，get_cluster_config以便装载新的集群分区配置及其他配置。

- plproxy.get_cluster_partitions(cluster_name text)

  此函数用于配置集群的水平分区信息

- plproxy.get_cluster_config(IN cluster_name text, out key text, out val text)

  此函数用于配置集群的超时参数、TCP的keepalive等连接参数。返回key-value对。如果值为0或NULL，表示使用默认值。

  它支持如下一些参数

  - connection_lifetime ：到后端数据库连接的最大存活时间。
  - query_timeout          ：如果查询请求在此时间内不返回，则此连接关闭。
  - disable_binary          ：不使用binary I/O方式连接后端数据库。
  - keepalive_idle           ：tcp keepalive参数
  - keepalive_interval    ：tcp keepalive参数
  - keepalive_count       ：tcp keepalive参数 

## cluster SQL/MED方式

```c
SQL/MED是sql语言中管理外部数据的一个扩展标准。MED:management of external data。它通过定义一个外部数据包装器和数据连接类型来管理外部数据。9.1开始提供对SQL/MED的支持，通过SQL/MED可以连接到各种异构数据库或其他PostgreSQL数据库。其相当于一套连接其他数据源的框架和标准。
在SQL/MED标准中，实现了一下四类数据库对象来访问外部数据源：
foreign data wrapper：外部数据包装器，FDW。相当于定义外部数据驱动
server：外部数据服务器，相当于定义一个外部数据源，需要制定外部数据源的FDW
user mapping：用户映射，主要把外部数据源的用户映射到本地用户，用于控制权限
foreign table：外部表，把外部数据源映射成数据库中的一张外部表
```

- 第一步穿件外部数据包装器

  ```sql
  create foreign data wrapper plproxy [validator plproxy_fdw_validator][options global options];
  在9.X版本及以上，命令如下
  create extension plproxy;
  ```

- 第二步定义一个cluster

  ```sql
  create server osdba_cluster01 foreign data wrapper plproxy
  options (
  	connection_lifetime '1800',
    p0 'dbname=datadb host=10.0.3.211',
    p1 'dbname=datadb host=10.0.3.212',
    p2 'dbname=datadb host=10.0.3.213',
    p3 'dbname=datadb host=10.0.3.214',
  ) 
  ```

- 第三步用户映射

  ```sql
  create user mapping for osdba server osdba_cluster01 options (user 'osdba',password 'xxxx');
  如果想为所用用户映射一个public映射，命令如下：
  create user mapping for public server osdba_cluster01 options (user 'osdba',password 'xxxx');
  ```

# plproxy api配置案例

| IP地址        | 角色           | 数据库名称 |
| ------------- | -------------- | ---------- |
| 192.168.56.31 | 代理数据库     | proxydb    |
| 192.168.56.32 | 后端数据节点01 | datadb     |
| 192.168.56.33 | 后端数据节点02 | datadb     |

## 代理节点安装plproxy

```shell
编译安装
sudo tar -xzvf plproxy-2.10.0.tar.gz
sudo chown -R postgres:postgres plproxy-2.10.0
make
make install
postgres@sc:/opt/pgsql14.2$ createdb proxydb
postgres@sc:/opt/pgsql14.2$ psql proxydb
psql (14.2)
Type "help" for help.
proxydb=# create extension plproxy;
CREATE EXTENSION
proxydb=#
```

## 所有节点修改认证

```shel
postgres@sc:/opt/pgsql14.2/data$ vi pg_hba.conf
#添加如下：允许192.168.56网段访问
#host    all             all             192.168.56.0/24            trust
```

## 代理节点定义集群信息

```sql
create schema plproxy;

create or replace function plproxy.get_cluster_version(cluster_name text) 
returns int4 
as $$
begin
	if cluster_name ='osdba_cluster01' then 
		return 1;
	end if ;
	raise exception 'unknown cluster';
end;
$$ language plpgsql;

create or replace function plproxy.get_cluster_partitions(cluster_name text) 
returns setof text 
as $$
begin
	if cluster_name ='osdba_cluster01' then 
		return next 'dbname=datadb host=192.168.56.32 user=postgres';
		return next 'dbname=datadb host=192.168.56.33 user=postgres';
		return;
	end if ;
	raise exception 'unknown cluster';
end;
$$ language plpgsql;

create or replace function plproxy.get_cluster_config(in cluster_name text,out key text,out val text) 
returns setof record 
as $$
begin
	 key:='connection_lifetime';
	 val:=30*60;
	 return next;
	 return;
end;
$$ language plpgsql;
```

## 数据节点创建测试表

```sql
create table users(
username text,
email text
);
```

## 代理库创建函数

```sql
create or replace function insert_user(i_username text,i_emailaddress text)
returns integer 
as $$
	cluster 'osdba_cluster01';
	run on hashtext(i_username);
$$ language plproxy;

create or replace function get_user_email(i_username text)
returns setof text 
as $$
	cluster 'osdba_cluster01';
	run on hashtext(i_username);
	select email from users where username=i_username;
$$ language plproxy;
```

## 数据节点创建函数

```sql
create or replace function insert_user(i_username text,i_emailaddress text)
returns integer 
as $$
	insert into users(username,email) values($1,$2);
	select 1;
$$ language sql;
```

## 测试执行

代理节点

```sql
postgres@sc:/etc$ psql proxydb
psql (14.2)
Type "help" for help.
proxydb=# select insert_user('张三','zs@163.com');
 insert_user
-------------
           1
(1 row)
proxydb=# select insert_user('李四','ls@163.com');
 insert_user
-------------
           1
(1 row)
proxydb=# select insert_user('wr','wr@163.com');
 insert_user
-------------
           1
(1 row)
proxydb=# select get_user_email('wr');
 get_user_email
----------------
 wr@163.com
(1 row)
proxydb=#
```

数据节点01

```sql
datadb=# select * from users;
 username |   email
----------+------------
 张三     | zs@163.com
 李四     | ls@163.com
(2 rows)

datadb=#
```

数据节点02

```sql
datadb=# select * from users;
 username |   email
----------+------------
 wr       | wr@163.com
(1 row)

datadb=#
```

