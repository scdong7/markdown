# postgresql 8.1.4简单安装

sudo systemctl set-default multi-user.target

dpkg -s libreadline6-dev

sudo service lightdm restart

sudo apt-get install libreadline6-dev

sudo apt-get install zlib1g-dev

1. wget https://ftp.postgresql.org/pub/source/v8.1.4/postgresql-8.1.4.tar.bz2
2. tar xjvf postgresql*.bz2 #解压至一个目录
3. cd potgresql-8.1.4
4. ./configure --prefix=/opt/pgsql8.14 --enable-debug CFLAGS="-O0 -g" --without-zlib#拟安装至/opt/pgsql8.14
5. 将"src/Makefile.global"中代码优化选项移除

   ```c
   // 移除前
   CFLAGS = -Wall (…) -fexcess-precision=standard -g -O2
   CXXFLAGS = -Wall (…) -g -O2
   // 移除后
   CFLAGS = -Wall (…) -fexcess-precision=standard -g
   CXXFLAGS = -Wall (…) -g
   ```
6. make
7. make install
8. adduser postgres #增加新用户，系统提示要给定新用户密码
9. mkdir /opt/pgsql8.14/data #创建数据库目录
10. chown -R postgres:postgres /opt/pgsql8.14/data
11. su - postgres #使用postgres帐号操作
12. /opt/pgsql8.14/bin/initdb -D /opt/pgsql8.14/data #初始化数据库
13. /opt/pgsql8.14/bin/pg_ctl -D /opt/pgsql8.14/data -l logfile start #启动数据库
14. /opt/pgsql8.14/bin/createdb genericdb #假定数据库名为gerericdb)
15. /opt/pgsql8.14/bin/psql genericdb # (进入数据库内部)
16. select pg_backend_pid();
17. 配置环境变量

    ```shell
    PG_HOME=/opt/pgsql8.14
    PGDATA=$PG_HOME/data
    LD_LIBRARY_PATH=$PG_HOME/lib:$LD_LIBRARY_PATH
    PATH=$PG_HOME/bin:$PATH

    export PG_HOME PGDATA LD_LIBRARY_PATH PATH
    ```
18. 配置vscode launch.json

    {
    // 使用 IntelliSense 了解相关属性。
    // 悬停以查看现有属性的描述。
    // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
    {
    "name": "launch启动",
    "type": "cppdbg",
    "request": "launch",
    "program": "/opt/pgsql8.14/bin/postgres",
    "args": [
    "-D","/opt/pgsql8.14/data"
    ],
    "stopAtEntry": false,
    "cwd": "${workspaceFolder}",
    "environment": [],
    "externalConsole": false,
    "MIMode": "gdb",
    "setupCommands": [
    {
    "description": "为 gdb 启用整齐打印",
    "text": "-enable-pretty-printing",
    "ignoreFailures": true
    },
    {
    "description":  "将反汇编风格设置为 Intel",
    "text": "-gdb-set disassembly-flavor intel",
    "ignoreFailures": true
    }
    ,
    {
    "text": "-gdb-set follow-fork-mode child",
    "ignoreFailures": true
    },
    {
    "text": "-gdb-set detach-on-fork on",
    "ignoreFailures": true
    }
    ]
    },
    {
    "name": "attach启动",
    "type": "cppdbg",
    "request": "attach",
    "program": "/opt/pgsql8.14/bin/postgres",
    "processId": "${command:pickProcess}",
    "MIMode": "gdb",
    "setupCommands": [
    {
    "description": "Enable pretty-printing for gdb",
    "text": "-enable-pretty-printing",
    "ignoreFailures": true
    },
    {
    "description":  "Set Disassembly Flavor to Intel",
    "text": "-gdb-set disassembly-flavor intel",
    "ignoreFailures": true
    }
    ]
    }
    ]
    }

    ```

    ```
