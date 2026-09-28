# postgresql 8.1.4简单安装

sudo systemctl set-default multi-user.target

dpkg -s libreadline6-dev

sudo service lightdm restart

sudo apt-get install libreadline6-dev

sudo apt-get install zlib1g-dev

1. wget https://ftp.postgresql.org/pub/source/v8.1.4/postgresql-8.1.4.tar.bz2
2. tar xjvf postgresql*.bz2 #解压至一个目录
3. cd potgresql-8.1.4
4. ./configure --prefix=/opt/pgsql8.14 --enable-debug CFLAGS="-O0 -g" --without-zlib --without-icu --without-openssl --without-libxml --without-libxslt --without-perl --without-python.    #拟安装至/opt/pgsql8.14
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
12. /opt/pgsql8.14/bin/initdb -D /opt/pgsql8.14/data -U postgres #初始化数据库
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
