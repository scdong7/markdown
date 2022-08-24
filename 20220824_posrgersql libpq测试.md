# 安装libpq库

sudo apt-get install libpq-dev 

# 编写测试代码

```c
#include <stdio.h>
#include <stdlib.h>
#include "libpq-fe.h"
/*
 * testlibpq.c
 *  Test the C version of LIBPQ, the POSTGRES frontend library.
 */
static void exit_nicely(PGconn *conn){
 PQfinish(conn);
 exit(EXIT_SUCCESS);
}

int main(){
 char    *pghost,
         *pgport,
         *pgoptions,
         *pgtty;
 char    *dbName;
 int   nFields;
 int   i,
       j;
 PGconn    *conn;
 PGresult   *res;
 /*
  * Begin by setting the parameters for a backend connection.
  * If the parameters are NULL, the system tries to use 
  * reasonable defaults by looking up environment variables 
  * or, failing that, using hardwired constants.
  */
 const char *conninfo="postgresql://postgres:postgres@localhost:5432/postgres";
 /* make a connection to the database */
 //conn = PQsetdb(pghost, pgport, pgoptions, pgtty, dbName);
 conn = PQconnectdb(conninfo);
 /* check to see that the backend connection was successfully made */
 if (PQstatus(conn) == CONNECTION_BAD)
 {
  fprintf(stderr, "Connection to database '%s' failed.\n", dbName);
  fprintf(stderr, "%s", PQerrorMessage(conn));
  exit_nicely(conn);
 }
 /* start a transaction block */
 res = PQexec(conn, "BEGIN");
 if (PQresultStatus(res) != PGRES_COMMAND_OK)
 {
  fprintf(stderr, "BEGIN command failed\n");
  PQclear(res);
  exit_nicely(conn);
 }
 /*
  * should PQclear PGresult whenever it is no longer needed 
  * so as to avoid memory leaks
  */
 PQclear(res);

 /*
  * fetch instances from the pg_database, the system catalog of
  * databases
  */
 res = PQexec(conn, "DECLARE myportal CURSOR FOR select * from pg_database");
 if (PQresultStatus(res) != PGRES_COMMAND_OK)
 {
  fprintf(stderr, "DECLARE CURSOR command failed\n");
  PQclear(res);
  exit_nicely(conn);
 }
 PQclear(res);

 res = PQexec(conn, "FETCH ALL in myportal");
 if (PQresultStatus(res) != PGRES_TUPLES_OK)
 {
  fprintf(stderr, "FETCH ALL command didn't return tuples properly\n");
  PQclear(res);
  exit_nicely(conn);
 }

 /* first, print out the attribute names */
 nFields = PQnfields(res);
 for (i = 0; i < nFields; i++)
  printf("%-15s", PQfname(res, i));

 printf("\n\n");

 /* next, print out the instances */
 for (i = 0; i < PQntuples(res); i++)
 {
  for (j = 0; j < nFields; j++)
   printf("%-15s", PQgetvalue(res, i, j));
  printf("\n");
 }

 PQclear(res);

 /* close the portal */
 res = PQexec(conn, "CLOSE myportal");
 PQclear(res);

 /* end the transaction */
 res = PQexec(conn, "END");
 PQclear(res);

 /* close the connection to the database and cleanup */
 PQfinish(conn);
 return 0;
}
```

# 编译测试

gcc -I/usr/include/postgresql -o testlibpq testlibpq.c -lpq

```shell
postgres@sc:/opt/clang$ ./testlibpq
oid            datname        datdba         encoding       datcollate     datctype       datistemplate  datallowconn   datconnlimit   datlastsysoid  datfrozenxid   datminmxid     dattablespace  datacl

13008          postgres       10             6              en_US.UTF-8    en_US.UTF-8    f              t              -1             13007          726            1              1663
1              template1      10             6              en_US.UTF-8    en_US.UTF-8    t              t              -1             13007          726            1              1663           {=c/postgres,postgres=CTc/postgres}
13007          template0      10             6              en_US.UTF-8    en_US.UTF-8    t              f              -1             13007          726            1              1663           {=c/postgres,postgres=CTc/postgres}
```

