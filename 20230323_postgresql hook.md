

# 1. 前言

**hook机制**来源于Windows平台。**hook**俗称**钩子** ,是Windows 消息的拦截机制。

每当特定的消息发出，在没有到达目的窗口前，钩子程序就先截获该消息。这时钩子函数就可以加工处理（改变该消息），也可以强制结束消息的传递（屏蔽消息）.因为系统钩子会影响系统中所有的应用程序，所以钩子函数必须放在独立的动态链接库 (DLL) 中。

hook机制，提供了一种修改PostgreSQL内核功能却不必修改内核代码的手段，且可以轻松的加载和还原。

在PG中也可以把hook看作是一种内置的过滤器，但他是个空壳，过滤器里什么都没有，可以实现这个hook，也就是往过滤器里加东西。

PostgreSQL使用HOOK时，根据HOOK的名字找到对应的定义，通常是HOOK被定义时，将跳转到对应的代码执行用户在插件中植入的代码片段.

# 2. 内部机制

## 2.1 PG 中 hook 如何工作

Hooks 存在一个全局函数指针，并初始为 NULL，表示hook未被使用。当PostgreSQL执行到hook处的时候，检查这个全局函数指针，

如果hook被设置为某个函数的地址时，转而执行新增的函数；否则执行标准函数

## 2.2 如何设置函数指针

hook 函数定义在共享库中。在加载共享库时，PG 会调用共享库中的 `_PG_init()` 函数。通过 `_PG_init` 函数设置函数指针（一般会把原来的指针保存）。

> 可以通过LOAD命令来实现动态加载共享库文件，也可以通过修改session_preload_libraries 或 shared_preload_libraries 参数来加载

## 2.3 如何清除函数指针

在卸载时，PG 调用共享库中的 `_PG_fini()` 函数

`_PG_fini` 函数需要重新设置函数指针（通常将原来的指针恢复）。

###这里要提一句，PG15版本自封装的扩展已经不包含 _PG_fini() 函数。且在官方文档里也指出：

```
a dynamically loaded file can contain an initialization function. If the file includes a function named _PG_init, that function will be called immediately after loading the file. The function receives no parameters and should return void. There is presently no way to unload a dynamically loaded file.
目前没有办法卸载动态加载的文件。所以在编写函数文件时_PG_fini() 函数可以忽略
```

# 3. 举例说明

* 需求: 删除数据库foodb时，只能使用foo用户删除，其他用户（包括超级用户）均不能删除此数据库。
* 思路: 删除数据库的命令是drop database，属于Utility命令，而PG提供了ProcessUtility_hook钩子可供使用

## 3.1 建立contrib目录：

```
[root@server contrib]# pwd
/soft/postgresql-9.1./contrib
[root@server contrib]# mkdir dbrestrict
[root@server contrib]# cd dbrestrict
```

## 3.2 代码实现

```c
/*
 * dbrestrict.c
 * Restrict drop of a given database to a given super-superuser only.
 * Under license "do whatever you want with that"
 */
#include "postgres.h"
#include "miscadmin.h"
#include "tcop/utility.h"

PG_MODULE_MAGIC;

// 声明函数
void _PG_init(void);
void _PG_fini(void);

static char *undroppabledb = "foodb";
static char *supersuperuser = "foo";

//全局变量初始化为NULL
static ProcessUtility_hook_type prev_utility_hook = NULL;

//声明函数
static void dbrestrict_utility(PlannedStmt *pstmt, const char *queryString,bool readOnlyTree,
                          ProcessUtilityContext context, ParamListInfo params,
                          QueryEnvironment *queryEnv,
                          DestReceiver *dest, QueryCompletion *qc);

/*
 * Personal process for DB drop restriction
 */
 
static void dbrestrict_utility(PlannedStmt *pstmt, const char *queryString,bool readOnlyTree,
                          ProcessUtilityContext context, ParamListInfo params,
                          QueryEnvironment *queryEnv,
                          DestReceiver *dest, QueryCompletion *qc)
{
	/* Do our custom process on drop database */
	switch (nodeTag(pstmt->utilityStmt))
	{
			case T_DropdbStmt:
			{
					DropdbStmt *stmt = (DropdbStmt *) pstmt->utilityStmt;
					char *username = GetUserNameFromId(GetUserId(),false);

					/*
					 * Check that only the authorized superuser foo can
					 * drop the database undroppable_foodb.
					 */
					if (strcmp(stmt->dbname, undroppabledb) == 0 &&
							strcmp(username, supersuperuser) != 0)
							ereport(ERROR,
									 (errcode(ERRCODE_INSUFFICIENT_PRIVILEGE),
									   errmsg("Only super-superuser \"%s\" can drop database \"%s\"",
													 supersuperuser, undroppabledb)));
					break;
			}
			default:
					break;
	}
    /*
     * Fallback to normal process, be it the previous hook loaded
     * or the in-core code path if the previous hook does not exist.
     */
    if (prev_utility_hook)
        (*prev_utility_hook) (pstmt, queryString, readOnlyTree, context, params, queryEnv, dest, qc);
    else
        standard_ProcessUtility(pstmt, queryString, readOnlyTree, context, params, queryEnv, dest, qc);
}

/*
 * _PG_init
 * Install the hook.
 */
void
_PG_init(void)
{
        prev_utility_hook = ProcessUtility_hook;
        ProcessUtility_hook = dbrestrict_utility;
}

/*
 * _PG_fini
 * Uninstall the hook.
 * 如果卸载则调用该函数，实际上是将ProcessUtility_hook赋回原值
 */
void
_PG_fini(void)
{
        ProcessUtility_hook = prev_utility_hook;
}
```

## 3.3 Makefile文件

```makefile
# contrib/dbrestrict/Makefile
MODULES = dbrestrict
OBJS = dbrestreict.s
# postgres build stuff
ifdef USE_PGXS
PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
else
subdir = contrib/dbrestrict
top_builddir = ../..
include $(top_builddir)/src/Makefile.global
include $(top_srcdir)/contrib/contrib-global.mk
endif
```

## 3.4 编译和安装

make & make install  生成 .so 文件并拷贝到lib文件

```shell
 [root@server dbrestrict]# make
gcc -O2 -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Wendif-labels -Wformat-security -fno-strict-aliasing -fwrapv -fpic -I. -I. -I../../src/include -D_GNU_SOURCE   -c -o dbrestrict.o dbrestrict.c
gcc -O2 -Wall -Wmissing-prototypes -Wpointer-arith -Wdeclaration-after-statement -Wendif-labels -Wformat-security -fno-strict-aliasing -fwrapv -fpic -L../../src/port  -Wl,-rpath,'/usr/local/pgsql/lib',--enable-new-dtags  -shared -o dbrestrict.so dbrestrict.o
rm dbrestrict.o
[root@server dbrestrict]# ls
Makefile  dbrestrict.c  dbrestrict.so
[root@server dbrestrict]# make install
/bin/mkdir -p '/usr/local/pgsql/lib'
/bin/sh ../../config/install-sh -c -m   dbrestrict.so '/usr/local/pgsql/lib/'
[root@server dbrestrict]# 
```

## 3.5 配置postgresql.conf文件

   把编译好的名为dbrestrict的动态库，追加到 shared_preload_libraries中或如后面load 命令加载共享库。

```shell
[postgres@server data]$ vim postgresql.conf
[postgres@server data]$ cat postgresql.conf | grep 'preload'
shared_preload_libraries = 'dbrestrict'         # (change requires restart)
#local_preload_libraries = ''
[postgres@server data]$ 
```

## 3.6 测试验证

```pgsql
-- 创建测试数据库
postgres=# create database foodb ;
CREATE DATABASE
--- 加载前：成功 drop
postgres=# drop database foodb ;
DROP DATABASE
--- 动态加载共享库文件：dbrestrict.so
postgres=# load 'dbrestrict.so';
LOAD
-- 创建测试数据库
postgres=# create database foodb ;
CREATE DATABASE
--- drop 失败
postgres=# drop database foodb ;
ERROR:  Only super-superuser "foo" can drop database "foodb"
-- 创建带有superuser属性的foo用户
create user foo superuser ;
--- drop 成功
postgres=# \c - foo
You are now connected to database "postgres" as user "foo".
postgres=# drop database foodb ;
DROP DATABASE

--# 需求测试完成！
```
