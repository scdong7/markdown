#  postgresql 8.1.4简单安装

dpkg -s libreadline6-dev

sudo service lightdm restart

sudo apt-get install libreadline6-dev

sudo apt-get install zlib1g-dev 

1. wget https://ftp.postgresql.org/pub/source/v8.1.4/postgresql-8.1.4.tar.bz2 
2. tar xjvf postgresql*.bz2 #解压至一个目录
3. cd potgresql-8.1.4
4. ./configure --prefix=/opt/pgsql --enable-debug CFLAGS="-O0" --without-zlib#拟安装至/opt/pgsql
5. make
6. make install
7. adduser postgres #增加新用户，系统提示要给定新用户密码
8. mkdir /opt/pgsql/data #创建数据库目录
9. chown -R postgres:postgres /opt/pgsql/data
10. su - postgres #使用postgres帐号操作
11. /opt/pgsql/bin/initdb -D /opt/pgsql/data #初始化数据库
12. /opt/pgsql/bin/pg_ctl -D /opt/pgsql/data -l logfile start #启动数据库
13. /opt/pgsql/bin/createdb genericdb #假定数据库名为gerericdb)
14. /opt/pgsql/bin/psql genericdb # (进入数据库内部)

15. select pg_backend_pid();