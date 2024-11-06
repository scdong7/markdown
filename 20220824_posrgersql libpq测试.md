# 安装libpq库

* ubuntu sudo apt-get install libpq-dev
* centos yum install postgresql-devel

# 编写测试代码

```c
/*
yum install postgresql-devel
gcc -I/usr/include/postgresql -o pg pg.c -lpq -std=c99
./pg user=postgres:postgres@localhost:5432/postgres query="select * from t3" head=on field=',' file=text.txt log=a.log --debug
*/
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <time.h>

#include "libpq-fe.h"

#define  MIN(a,b) ((a) > (b) ? (b) : (a))

#define MAJOR_VERSION_NUMBER       0
#define MINOR_VERSION_NUMBER       1

#define FEEDBACK       50000

typedef char text; 

struct PARAM {
   text name[64];
   text connstr[128];
   text query[128];
   text field[128];
   text file[128];
   text log[128];
   int header;
   int debug;
   
   int fileld_ln;

} ;
struct PARAM *param;
FILE *fp_log=NULL;

void print_help()
{
  printf("\n----------------------------------------------------------------------------\n");
  printf("- %s: release %d.%d \n",param->name,MAJOR_VERSION_NUMBER,MINOR_VERSION_NUMBER);
  printf("- author: sc \n"); 
  printf("----------------------------------------------------------------------------\n");
  printf(" Usage: %s keyword=value [,keyword=value,...]\n",param->name);
  printf(" Valid Keywords:\n");
  printf("       user     = username/password@tnsname\n");
  printf("       query    = select statement, can simply speicify a table name\n"); 
  printf("       field    = seperator string between fields\n"); 
  printf("       file     = output file name, default: uldrdata.txt\n");
  printf("       head     = print row header(Yes|No,ON|OFF,1|0)\n");
  printf("       log      = log file name\n");
  printf("\n");
 
}

int get_param(int argc,char **argv){
   char *proname = strrchr(argv[0], '/');
   char tempbuf[1024];
   if (proname) {
      proname=proname + 1;
   } else {
      proname=argv[0];
   }
   param=(struct PARAM *)malloc(sizeof(struct PARAM));  
   memset(param,0,sizeof(param));
   memcpy(param->file,"pg.txt",6);
   memcpy(param->name,proname,MIN(strlen(proname),64));
   if(argc>1){
      for(int i=1;i<argc;i++){
         if(strncasecmp("user=",argv[i],5)==0){
            memcpy(param->connstr,(text *)argv[i]+5,MIN(strlen(argv[i]) - 5,123)); 
         }
         else if(strncasecmp("query=",argv[i],6)==0){
            memcpy(param->query,(text *)argv[i]+6,MIN(strlen(argv[i]) - 6,122));
         }else if(strncasecmp("field=",argv[i],6)==0){
            memcpy(param->field,(text *)argv[i]+6,MIN(strlen(argv[i]) - 6,122));
            param->fileld_ln=strlen(param->field);
         }else if(strncasecmp("file=",argv[i],5)==0){
            memcpy(param->file,(text *)argv[i]+5,MIN(strlen(argv[i]) - 5,123));
         }else if(strncasecmp("log=",argv[i],4)==0){
            memcpy(param->log,(text *)argv[i]+4,MIN(strlen(argv[i]) - 4,124));
         }else if(strncasecmp("head=",argv[i],5)==0){
            memset(tempbuf,0,16);
            memcpy(tempbuf,(text *)argv[i]+5,MIN(strlen(argv[i]) - 5,123));
            if (strncasecmp(tempbuf,"YES",3) == 0 || strncasecmp(tempbuf,"ON",2) == 0  || strncasecmp(tempbuf,"1",1) == 0) 
               param->header = 1;
         }else if(strncasecmp("--debug",argv[i],7)==0){
            param->debug = 1;
         }else if(strncasecmp("-?",argv[i],2)==0) {
            print_help();
         }
      }
   }else{
      print_help();
   }
   if(param->debug)
   printf("param.name=%s,param.connstr=%s,param.table=%s,param.header=%d,param.field=%s,param.file=%s,param.log=%s\n",(char *)param->name,(char *)param->connstr,(char *)param->query,param->header,
      (char *)param->field,(char *)param->file,(char *)param->log);
   return 0;
}

static void exit_nicely(PGconn *conn){
 PQfinish(conn);
 exit(EXIT_SUCCESS);
}

void print_feedback(int row)
{
  time_t now = time(0);
  struct tm *ptm = localtime(&now);
 
  
  fprintf((fp_log == NULL?stderr:fp_log),"%8u rows exported at %04d-%02d-%02d %02d:%02d:%02d\n",
        row,
	ptm->tm_year + 1900,
	ptm->tm_mon + 1,
	ptm->tm_mday,
	ptm->tm_hour,
	ptm->tm_min,
	ptm->tm_sec);
  fflush((fp_log == NULL?stderr:fp_log));
}

int get_attr(PGresult  *res){
   int  nFields; 
   int  nTuples;
   FILE *fp=NULL;
   fp=fopen(param->file,"wb");
   /* first, print out the attribute names */
   nFields = PQnfields(res);
   if(param->header){
      for (int i = 0; i < nFields; i++){
         fprintf(fp,"%s",PQfname(res, i));
         if(i<nFields-1){
            fwrite(param->field,1,1,fp);
         }
      }
      fprintf(fp,"\n");
   }
    /* next, print out the instances */
   nTuples=PQntuples(res);
   for (int i = 0; i < nTuples; i++)
   {
      for (int j = 0; j < nFields; j++){
         fprintf(fp,"%s",PQgetvalue(res, i, j));
         if(j<nFields-1){
            fwrite(param->field,param->fileld_ln,1,fp);
         }
   
      }
      fprintf(fp,"\n");
      if((i+1)%FEEDBACK==0){
         print_feedback(FEEDBACK);
      }
   }
   if(nTuples%FEEDBACK != 0 ){
         print_feedback(nTuples%FEEDBACK);
   }
   fclose(fp);
   PQclear(res); 
   return nTuples;
}

int main(int argc,char *argv[]){
   char conn_pre[]="postgresql://"; 
   PGconn    *conn;
   PGresult  *res; 
   int rows;

   get_param(argc,argv);

   fp_log=fopen(param->log,"wb");
   fprintf(fp_log==NULL?stderr:fp_log, "%s export tools is working\n",param->name);
   conn = PQconnectdb(strcat(conn_pre,(char *)param->connstr));
   if (PQstatus(conn) == CONNECTION_BAD)
   {
      fprintf(fp_log==NULL?stderr:fp_log, "Connection to database failed.\n");
      fprintf(fp_log==NULL?stderr:fp_log, "%s", PQerrorMessage(conn));
      exit_nicely(conn);
   }
   /* start a transaction block */
   res = PQexec(conn, "BEGIN");
   if (PQresultStatus(res) != PGRES_COMMAND_OK)
   {
   fprintf(fp_log==NULL?stderr:fp_log, "BEGIN command failed\n");
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
   res = PQexec(conn, (char *)param->query);
   if (PQresultStatus(res) != PGRES_TUPLES_OK)
   {
      fprintf(fp_log==NULL?stderr:fp_log, PQresultErrorMessage(res));
      PQclear(res);
      exit_nicely(conn);
   }
   
   rows=get_attr(res);

   /* end the transaction */
   res = PQexec(conn, "END");
   PQclear(res);

   /* close the connection to the database and cleanup */
   PQfinish(conn); 

   fprintf(fp_log==NULL?stderr:fp_log, "output file %s closed at %d rows.\n",param->file,rows);
   
   if(fp_log)
      fclose(fp_log);
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
