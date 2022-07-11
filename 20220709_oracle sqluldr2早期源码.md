# readme

```c
A tool for exporting data from Oracle to csv.
Access http://www.anysql.net/ for SQLULDR2(company version with free, No source code)

ociuldr.c	# origin version


tbuldr.c	# a modify version
----------------------------------------------------------------------------
- tbuldr: Release 2.21 (ociuldr)
- First issued by Lou Fangxin (楼方鑫) Copyright 2004/2008, all rights reserved.
- Modified by Ning Haiyuan (http://www.NinGoo.net).
----------------------------------------------------------------------------
 Usage: TBULDR keyword=value [,keyword=value,...]
 Valid Keywords:
       user     = username/password@tnsname
       query    = select statement, can simply speicify a table name
       sql      = SQL file name
       field    = seperator string between fields
       record   = seperator string between records
       enclose  = fields enclose string
       file     = output file name, default: uldrdata.txt
       head     = print row header(Yes|No,ON|OFF,1|0)
       read     = set DB_FILE_MULTIBLOCK_READ_COUNT at session level
       sort     = set SORT_AREA_SIZE at session level (UNIT:MB)
       hash     = set HASH_AREA_SIZE at session level (UNIT:MB)
       serial   = set _serial_direct_read to TRUE if 1 at session level
       trace    = set event 10046 to given level at session level
       table    = table name in the sqlldr control file
       mode     = sqlldr option, INSERT or APPEND or REPLACE or TRUNCATE
       log      = log file name, prefix with + to append mode
       long     = maximum long field size, default 8192 max 65534
       array    = array fetch size, default 50
       buffer   = sqlldr READSIZE and BINDSIZE, default 16 (MB)
       feedback = display progress every x rows, default 500000
       form     = display rows as form (Yes|No)
       fixlen   = fix length format (Yes|No)

  for field, record and enclose, use '0x' to specify hex character code
  r=0x0d n=0x0a |=0x7c ,=0x2c t=0x09  (more: man ascii)
```

# ociuldr.c

```c
#ifdef RCSID
static char *RCSid =
   "$Header: ociuldr.c 2005.05.19 Lou Fangxin, http://www.anysql.net $ ";
#endif /* RCSID */

/*
   NAME
     ociuldr.c - Using OCI to rewrite the unload script.
   MODIFIED   (MM/DD/YY)
    Lou Fangxin    2005.05.19 -  Initial write.
    Lou Fangxin    2005.05.22 -  Add File Option to command
    Lou Fangxin    2005.05.25 -  Enable login as sysdba
*/

#include <stdio.h>
#include <ctype.h>
#include <string.h>
#include <stdlib.h>
#include <time.h>

#include <oratypes.h>
#include <ocidfn.h>
#ifdef __STDC__
#include <ociapr.h>
#else
#include <ocikpr.h>
#endif
#include <ocidem.h>

#include <oci.h>

/* Constants used in this program. */
#define MAX_SELECT_LIST_SIZE    1023
#define MAX_ITEM_BUFFER_SIZE    33
#define PARSE_NO_DEFER           0
#define PARSE_V7_LNG             2
#define MAX_BINDS               12
#define MAX_SQL_IDENTIFIER      31
#define DEFAULT_BUFFER_SIZE     524288
#define ROW_BATCH_SIZE          500000
#define UNLOAD_BUFFER_SIZE      2097152

#if defined(_WIN32)
#define STRNCASECMP memicmp
#else
#define STRNCASECMP strncasecmp
#endif

#if !defined(MIN)
#define  MIN(a,b) ((a) > (b) ? (b) : (a))
#endif

#if !defined(MAX)
#define  MAX(a,b) ((a) < (b) ? (b) : (a))
#endif

struct COLUMN
{
    /* Describe */
    sb4             dbsize;
    sb2             dbtype;
    sb1             buf[MAX_ITEM_BUFFER_SIZE];
    sb4             buflen;
    sb4             dsize;
    sb2             precision;
    sb2             scale;
    sb2             nullok;

    /*+ Fetch */
    ub1             *colbuf;
    sb2             *indp;
    ub2             *col_retlen;
    ub2             *col_retcode;

    /*+ Point to next column */
    struct          COLUMN *next;
    char fmtstr[64];
};

ub4     DEFAULT_ARRAY_SIZE =   50;
ub4     DEFAULT_LONG_SIZE = 32768;
ub4     hda[HDA_SIZE/(sizeof(ub4))];

FILE *fp_log = NULL;
int return_code = 0;

sword LogonDB(Lda_Def *lda,text *username, text *password,ub4 mode)
{
    sword n;
 
    if (olog(lda, (ub1 *)hda, username, -1, password, -1,
                 (text *) 0, -1, mode))
    {
        fprintf((fp_log == NULL?stdout:fp_log),"Cannot connect as %s.\n", username);
    }
    else
    {
        return 0;
    }
    fprintf((fp_log == NULL?stdout:fp_log),"Connection failed.  Exiting...\n");
    return -1;
}

void SQLError(Lda_Def *lda, Cda_Def *cda)
{
    text msg[512];
    sword n;

    n = oerhms(lda, cda->rc, msg, (sword) sizeof (msg));
    fprintf((fp_log == NULL?stdout:fp_log),"%.*s", n, msg);
}

sword CreateSQL(Lda_Def *lda, Cda_Def *cda)
{
    if (oopen(cda, lda, (text *) 0, -1, -1, (text *) 0, -1))
    {
        fprintf((fp_log == NULL?stdout:fp_log),"Error opening cursor [OOPEN].\n");
        return -1;
    }
    return 0;
}

sword ParseSQL(Lda_Def *lda, Cda_Def *cda, text *sql_statement)
{
    if (oparse(cda, (text *) sql_statement, (sb4) -1,
               (sword) PARSE_NO_DEFER, (ub4) PARSE_V7_LNG))
    {
        SQLError(lda,cda);
        return -1;
    }
    return 0;
}

sword BindValue(Lda_Def *lda, Cda_Def *cda, text *vname, ub1 *val)
{

  if (obndrv(cda, vname, -1, 
            (ub1 *)val,  -1,
            VARCHAR2_TYPE,-1, (sb2 *) 0, (text *) 0, -1, -1))
  {
     SQLError(lda,cda);
     return -1;
  }
  return 0;
}

sword executeStatement(Lda_Def *lda, Cda_Def *cda)
{
     if (oexec(cda))
     {
        SQLError(lda,cda);
        return -1;
     }
     return 0;
}

sword getColumns(FILE *fpctl, Lda_Def *lda,Cda_Def *cda, struct COLUMN *collist, int fixlen)
{
    int totallen=1;
    sword col;
    struct COLUMN *tempcol;
    struct COLUMN *nextcol;
    ub1 *buf;

    nextcol = collist;
 
    /* Describe the select-list items. */
    if(fpctl != NULL) fprintf(fpctl,"(\n");
    for (col = 0; col < MAX_SELECT_LIST_SIZE; col++)
    {
        tempcol = (struct COLUMN *) malloc(sizeof(struct COLUMN));
        tempcol-> indp        = (sb2 *)malloc(DEFAULT_ARRAY_SIZE * sizeof(sb2));
        tempcol-> col_retlen  = (ub2 *)malloc(DEFAULT_ARRAY_SIZE * sizeof(ub2));
        tempcol-> col_retcode = (ub2 *)malloc(DEFAULT_ARRAY_SIZE * sizeof(ub2));

        tempcol->next = NULL;
        tempcol->colbuf = NULL;
			  memset(tempcol->fmtstr,0,64);

        tempcol->buflen = MAX_ITEM_BUFFER_SIZE;
        if (odescr(cda, col + 1, &(tempcol->dbsize),
                   &(tempcol->dbtype), &(tempcol->buf[0]),
                   &(tempcol->buflen), &(tempcol->dsize),
                   &(tempcol->precision), &(tempcol->scale),
                   &(tempcol->nullok)))
        {
            if(fpctl != NULL) fprintf(fpctl,"\n");
            free(tempcol);
            /* Break on end of select list. */
            if (cda->rc == VAR_NOT_IN_LIST)
                break;
            else
            {
                SQLError(lda,cda);
                return -1;
            }
        }

        if(col) 
        {
          if(fpctl != NULL) fprintf(fpctl,",\n");
        }

        nextcol->next = tempcol;
        nextcol=tempcol;

        nextcol->buf[nextcol->buflen]='\0';
 
        switch(nextcol->dbtype)
        {
            case DATE_TYPE:
                nextcol->dsize=19;
                nextcol->dbtype=(fixlen?SQLT_AVC:STRING_TYPE);
                if(fpctl != NULL)
                {
                   if (fixlen)
                      fprintf(fpctl,"  %s POSITION(%d:%d) DATE \"YYYY-MM-DD HH24:MI:SS\"", 
                          nextcol->buf , totallen, totallen + nextcol->dsize-1);
                   else
                      fprintf(fpctl,"  %s DATE \"YYYY-MM-DD HH24:MI:SS\"", nextcol->buf );
                }
                break;
            case 180: /* TIMESTAMP */
                nextcol->dsize=26;
                nextcol->dbtype=(fixlen?SQLT_AVC:STRING_TYPE);
                if(fpctl != NULL)
                { 
                   if (fixlen)
                      fprintf(fpctl,"  %s POSITION(%d:%d) TIMESTAMP \"YYYY-MM-DD HH24:MI:SSXFF\"",
                           nextcol->buf , totallen, totallen + nextcol->dsize-1);
                   else
                      fprintf(fpctl,"  %s TIMESTAMP \"YYYY-MM-DD HH24:MI:SSXFF\"", nextcol->buf );
                }
                break;
            case 181: /* TIMESTAMP WITH TIMEZONE */
                nextcol->dsize=33;
                nextcol->dbtype=(fixlen?SQLT_AVC:STRING_TYPE);
                if(fpctl != NULL)
                {
                   if (fixlen)
                       fprintf(fpctl,"  %s POSITION(%d:%d) TIMESTAMP WITH TIME ZONE \"YYYY-MM-DD HH24:MI:SSXFF TZH:TZM\"", 
                           nextcol->buf , totallen, totallen + nextcol->dsize-1);
                   else
                       fprintf(fpctl,"  %s TIMESTAMP WITH TIME ZONE \"YYYY-MM-DD HH24:MI:SSXFF TZH:TZM\"", nextcol->buf );
                }
                break;
            case 24:  /* LONG RAW */
            case 113: /* BLOB */
               nextcol->dsize=DEFAULT_LONG_SIZE;
               nextcol->dbtype=(fixlen?SQLT_AVC:STRING_TYPE);
               if(fpctl != NULL)
               {
                  if (fixlen)
                      fprintf(fpctl,"  %s POSITION(%d:%d) CHAR(%d) ", 
                          nextcol->buf, totallen, totallen + nextcol->dsize-1, DEFAULT_LONG_SIZE);
                  else
                      fprintf(fpctl,"  %s CHAR(%d) ", nextcol->buf, DEFAULT_LONG_SIZE);
               }
               break;
            case ROWID_TYPE:
                nextcol->dsize=18;
                nextcol->dbtype=(fixlen?SQLT_AVC:STRING_TYPE);
                if(fpctl != NULL)
                {
                    if (fixlen)
                       fprintf(fpctl, "  %s POSITION(%d:%d) CHAR(%d)", 
                            nextcol->buf, totallen, totallen + nextcol->dsize-1, nextcol->dsize);
                    else
                       fprintf(fpctl, "  %s CHAR(%d)", nextcol->buf,nextcol->dsize);
                }
                break;
            case NUMBER_TYPE:
               if (nextcol->precision)
                   nextcol->dsize=nextcol->precision;
               else
                   nextcol->dsize=40;
               nextcol->dbtype=(fixlen?SQLT_AVC:STRING_TYPE);
               if(fpctl != NULL)
               {
                  if (fixlen)
                      fprintf(fpctl,"  %s POSITION(%d:%d) CHAR(%d)", 
                           nextcol->buf,totallen, totallen + nextcol->dsize-1, nextcol->dsize);
                  else
                      fprintf(fpctl,"  %s CHAR(%d)", nextcol->buf,nextcol->dsize);
               }
               break;
            case 112: /* CLOB */
            case 114: /* BFILE */
               nextcol->dsize=DEFAULT_LONG_SIZE;
               nextcol->dbtype=(fixlen?SQLT_AVC:STRING_TYPE);
               if(fpctl != NULL)
               {
                  if (fixlen)
                      fprintf(fpctl,"  %s POSITION(%d:%d) CHAR(%d)", 
                          nextcol->buf, totallen, totallen + nextcol->dsize-1, DEFAULT_LONG_SIZE);
                  else
                      fprintf(fpctl,"  %s CHAR(%d)", nextcol->buf,DEFAULT_LONG_SIZE);
               }
               break;
            default:
               nextcol->dbtype=(fixlen?SQLT_AVC:STRING_TYPE);
	             if (nextcol->dsize>4000) nextcol->dsize = 4000;
	             if (nextcol->dsize==0)  nextcol->dsize = 4000;
               if(fpctl != NULL)
               {
                  if (fixlen)
                     fprintf(fpctl,"  %s POSITION(%d:%d) CHAR(%d)", nextcol->buf, 
                            totallen, totallen + nextcol->dsize-1,
                            (nextcol->dsize==0?DEFAULT_LONG_SIZE:nextcol->dsize));
                  else
                     fprintf(fpctl,"  %s CHAR(%d)", nextcol->buf,(nextcol->dsize==0?DEFAULT_LONG_SIZE:nextcol->dsize));
               }
               break;
        }
        /* Set for long type column */
        if (nextcol->dsize > DEFAULT_LONG_SIZE || nextcol->dsize == 0)  
            nextcol->dsize = DEFAULT_LONG_SIZE;
        /* add one more byte to store the ternimal char of string */
        sprintf(nextcol->fmtstr,"%%-%ds", nextcol->dsize);
        totallen = totallen + nextcol->dsize;
        nextcol->dsize ++; 

        nextcol->colbuf = malloc(DEFAULT_ARRAY_SIZE * (nextcol->dsize));
        memset(nextcol->colbuf,0,DEFAULT_ARRAY_SIZE * (nextcol->dsize));

        if (odefin(cda, col + 1,
                   nextcol->colbuf, nextcol->dsize, nextcol->dbtype,
                   -1, nextcol->indp, (text *) 0, -1, -1,
                   nextcol->col_retlen,nextcol->col_retcode))
        {
            SQLError(lda,cda);
            return -1;
        }
    }
    if(fpctl != NULL)
       fprintf(fpctl,")\n");
    fputs("\n",(fp_log == NULL?stdout:fp_log));
    return col;
}

void  printRowInfo(ub4 row)
{
	time_t now = time(0);
	struct tm *ptm = localtime(&now);
	fprintf((fp_log == NULL?stdout:fp_log),"%8u rows exported at %04d-%02d-%02d %02d:%02d:%02d\n",
                row,
		ptm->tm_year + 1900,
		ptm->tm_mon + 1,
		ptm->tm_mday,
		ptm->tm_hour,
		ptm->tm_min,
		ptm->tm_sec);
        fflush((fp_log == NULL?stdout:fp_log));
}

FILE *openFile(const text *fname, text tempbuf[], int batch)
{
   FILE *fp=NULL;
   int i, j, len;
   time_t now = time(0);
   struct tm *ptm = localtime(&now);   
   
   len = strlen(fname);
   j = 0;
   for(i=0;i<len;i++)
   {
      if (*(fname+i) == '%')
      {
          i++;
	  if (i < len)
	  {
            switch(*(fname+i))
            {
              case 'Y':
              case 'y':
                j += sprintf(tempbuf+j, "%04d", ptm->tm_year + 1900);
								break;
              case 'M':
              case 'm':
                j += sprintf(tempbuf+j, "%02d", ptm->tm_mon + 1);
								break;
              case 'D':
              case 'd':
                j += sprintf(tempbuf+j, "%02d", ptm->tm_mday);
								break;
              case 'W':
              case 'w':
                j += sprintf(tempbuf+j, "%d", ptm->tm_wday);
								break;
              case 'B':
              case 'b':
                j += sprintf(tempbuf+j, "%d", batch);
								break;
              default:
                tempbuf[j++] = *(fname+i);
								break;
            }
          }
      }
      else
      {
         tempbuf[j++] = *(fname+i);
      }
   }
   tempbuf[j]=0;
   if (tempbuf[0] == '+')
       fp = fopen(tempbuf+1, "ab+");
   else
       fp = fopen(tempbuf, "wb+");
   return fp;
}

void printRow(text *fname, Lda_Def *lda,Cda_Def *cda,struct COLUMN *col, text *field, int flen,text *record, int rlen, int batch, int header, int dispfrm, int fixlen)
{
    int bcount=1,j=0;
    sword r,rows,colcount,c;
    struct COLUMN *p;
    ub4 trows;
    FILE *fp;
    text tempbuf[512];

    struct COLUMN *cols[MAX_SELECT_LIST_SIZE];

    trows=0;
    colcount = 0;

    p = col->next;
    while(p != NULL)
    {
        cols[colcount] = p;
        p=p->next;
        colcount ++;
    }

    memset(tempbuf,0,512);

    if((fp = openFile(fname,tempbuf,bcount)) == NULL) 
    {
       fprintf((fp_log == NULL?stdout:fp_log),"ERROR -- Cannot write to file : %s\n", tempbuf);
       return_code = 6;
       return;
    }

    if (header)
    {
       for(c=0;c<colcount;c++)
       {
          if (fixlen)
              fprintf(fp,cols[c]->fmtstr,cols[c]->buf);
          else
          {
              fprintf(fp,"%s",cols[c]->buf);
              if (c < colcount - 1) 
                 fwrite(field,flen,1,fp);
          }
       }
       fwrite(record,rlen,1,fp);
    }

    printRowInfo(trows);
    for (;;)
    {
        rows = DEFAULT_ARRAY_SIZE;
        if (ofen(cda,DEFAULT_ARRAY_SIZE))
        {
            if (cda->rc != NO_DATA_FOUND)
            {
                 return_code = 7;
                 SQLError(lda,cda);
            }
            rows = cda->rpc % DEFAULT_ARRAY_SIZE;
        }
        for(r=0;r<rows;r++)
        {
           for(c=0;c<colcount;c++)
           {
              if (dispfrm)
              {
                  fprintf(fp,"%-31s: ",cols[c]->buf);
              }
              if (*(cols[c]->indp+r) >= 0)
              {
                if (fixlen)
                    fprintf(fp,cols[c]->fmtstr, cols[c]->colbuf+(r* cols[c]->dsize));
                else
                   fwrite(cols[c]->colbuf+(r* cols[c]->dsize),*(cols[c]->col_retlen+r),1,fp);
              }
	            else
	            {
	      	      if (fixlen)
	      	      {
			           fprintf(fp,cols[c]->fmtstr, " ");
		        		}
	            }
              if (dispfrm)
              {
              	  fprintf(fp,"\n");
              }
              else
              {
                 if (c < colcount - 1 && fixlen == 0)
                 {
                      fwrite(field,flen,1,fp);
                 }
              }
           }
           fwrite(record, rlen, 1, fp);
           trows ++;
           if (trows % ROW_BATCH_SIZE  == 0)
           {
              printRowInfo(trows);
              if(batch && ((trows / ROW_BATCH_SIZE) % batch) == 0)
              {
                 fprintf((fp_log == NULL?stdout:fp_log),"         output file %s closed at %u rows.\n", tempbuf, trows);
                 fclose(fp);
                 bcount ++;
                 memset(tempbuf,0,512);
                 if((fp = openFile(fname,tempbuf,bcount)) == NULL) 
                 {
                   fprintf((fp_log == NULL?stdout:fp_log),"ERROR -- Cannot write to file : %s\n", tempbuf);
                   return_code = 6;
                   return;
                 }
                 if (header)
                 {
                    for(c=0;c<colcount;c++)
                    {
                       if (fixlen)
                           fprintf(fp,cols[c]->fmtstr,cols[c]->buf);
                       else
                       {
                           fprintf(fp,"%s",cols[c]->buf);
                           if (c < colcount - 1)
                              fwrite(field,flen,1,fp);
                       }
                    }
                    fwrite(record,rlen,1,fp);
                 }
                 trows = 0;
              }
           }
        }
        if (rows < DEFAULT_ARRAY_SIZE) break;
    }
    if (trows % ROW_BATCH_SIZE != 0)
       printRowInfo(trows);
    fclose(fp);
    fprintf((fp_log == NULL?stdout:fp_log),"         output file %s closed at %u rows.\n\n", tempbuf, trows);
    fflush((fp_log == NULL?stdout:fp_log));
}

void freeColumn(struct COLUMN *col)
{
   struct COLUMN *p,*temp;
   p=col->next;

   col->next = NULL;
   while(p!=NULL)
   {
     free(p->colbuf);
     free(p->indp);
     free(p->col_retlen);
     free(p->col_retcode);
     temp=p;
     p=temp->next;
     free(temp);
   }
}

ub1  getHexIndex(char c)
{
   if ( c >='0' && c <='9') return c - '0';
   if ( c >='a' && c <='f') return 10 + c - 'a';
   if ( c >='A' && c <='F') return 10 + c - 'A';
   return 0;
}

int convertOption(const ub1 *src, ub1* dst, int mlen)
{
   int i,len,pos;
   ub1 c,c1,c2;

   i=pos=0;
   len = strlen(src);
   
   
   while(i<MIN(mlen,len))
   {
      if ( *(src+i) == '0')
      {
          if (i < len - 1)
          {
             c = *(src+i + 1);
             switch(c)
             {
                 case 'x':
                 case 'X':
                   if (i < len - 3)
                   {
                       c1 = getHexIndex(*(src+i + 2));
                       c2 = getHexIndex(*(src+i + 3));
                       *(dst + pos) = (ub1)((c1 << 4) + c2);
                       i=i+2;
                   }
                   else if (i < len - 2)
                   {
                       c1 = *(src+i + 2);
                       *(dst + pos) = c1;
                       i=i+1;
                   }
                   break;
                 default:
                   *(dst + pos) = c;
                   break;
             }
             i = i + 2;
          }
          else
          {
            i ++;
          }
      }
      else
      {
          *(dst + pos) = *(src+i);
          i ++;
      }
      pos ++;
   }
   *(dst+pos) = '\0';
   return pos;
}

int main(int argc, char *argv[])
{

    struct COLUMN col;

    sword n,i,argcount=0;
    Lda_Def conn;
    Cda_Def stmt;
    
    int  frmdisp=0;
    ub1 *iobuf;
    text tempbuf[512];
    text username[132]="";
    text ctlfname[256]="";
    text tabname[132]="";
    text tabmode[132]="INSERT";
    text query[32768]="";
    text field[132]=",";
    text logfile[256]="";
    text record[132]="\n";
    text sqlfname[255]="";
    text fname[255]="uldrdata.txt";
    text argname[128][20];
    ub1  argval [128][20];

    int  buffer= 16777216;
    int  hsize = 0;
    int  ssize = 0;
    int  bsize = 0;
    int  serial= 0;
    int  trace = 0;
    int  batch = 0;
    int  header= 0;

    ub4  lmode;

    int flen,rlen;

    FILE *fp;
    FILE *fpctl;

    flen = rlen = 1;
    col.next=NULL;
    lmode = OCI_LM_DEF;

    for(i=0;i<argc;i++)
    {
      if (STRNCASECMP("user=",argv[i],5)==0)
      {
          memset(username,0,132);
          memcpy(username,argv[i]+5,MIN(strlen(argv[i]) - 5,131));
      }
      else if (STRNCASECMP("query=",argv[i],6)==0)
      {
          memset(query,0,8192);
          memcpy(query,argv[i]+6,MIN(strlen(argv[i]) - 6,8191));
      }     
      else if (STRNCASECMP("field=",argv[i],6)==0)
      {
          memset(field,0,132);
          flen=convertOption(argv[i]+6,field,MIN(strlen(argv[i]) - 6,131));
      }     
      else if (STRNCASECMP("sql=",argv[i],4)==0)
      {
          memset(sqlfname,0,132);
          memcpy(sqlfname,argv[i]+4,MIN(strlen(argv[i]) - 4,254));
      }
      else if (STRNCASECMP("record=",argv[i],7)==0)
      {
          memset(record,0,132);
          rlen=convertOption(argv[i]+7,record,MIN(strlen(argv[i]) - 7,131));
      }     
      else if (STRNCASECMP("file=",argv[i],5)==0)
      {
          memset(fname,0,132);
          memcpy(fname,argv[i]+5,MIN(strlen(argv[i]) - 5,254));
      }     
      else if (STRNCASECMP("log=",argv[i],4)==0)
      {
          memset(logfile,0,256);
          memcpy(logfile,argv[i]+4,MIN(strlen(argv[i]) - 4,254));
      }     
      else if (STRNCASECMP("table=",argv[i],6)==0)
      {
          memset(tabname,0,132);
          memcpy(tabname,argv[i]+6,MIN(strlen(argv[i]) - 6,128));
      }     
      else if (STRNCASECMP("mode=",argv[i],5)==0)
      {
          memset(tabmode,0,132);
          memcpy(tabmode,argv[i]+5,MIN(strlen(argv[i]) - 5,128));
      }     
      else if (STRNCASECMP("head=",argv[i],5)==0)
      {
          memset(tempbuf,0,132);
          memcpy(tempbuf,argv[i]+5,MIN(strlen(argv[i]) - 5,128));
          header = 0;
          if (STRNCASECMP(tempbuf,"YES",3) == 0) header = 1;
          if (STRNCASECMP(tempbuf,"ON",3) == 0) header = 1;
      }     
      else if (STRNCASECMP("sort=",argv[i],5)==0)
      {
          memset(tempbuf,0,512);
          memcpy(tempbuf,argv[i]+5,MIN(strlen(argv[i]) - 5,254));
          ssize = atoi(tempbuf);
          if (ssize < 0) ssize = 0;
          if (ssize > 512) ssize = 512;
      }     
      else if (STRNCASECMP("buffer=",argv[i],7)==0)
      {
          memset(tempbuf,0,512);
          memcpy(tempbuf,argv[i]+7,MIN(strlen(argv[i]) - 7,254));
          buffer = atoi(tempbuf);
          if (buffer < 8) buffer = 8;
          if (ssize > 100) buffer = 100;
          buffer = buffer * 1048576;
      }    
      else if (STRNCASECMP("form=",argv[i],5)==0)
      {
          memset(tempbuf,0,132);
          memcpy(tempbuf,argv[i]+5,MIN(strlen(argv[i]) - 5,128));
          frmdisp=0;
          if (STRNCASECMP(tempbuf,"YES",3) == 0) frmdisp = 1;
          if (STRNCASECMP(tempbuf,"ON",3) == 0) frmdisp = 1;
      }       
      else if (STRNCASECMP("long=",argv[i],5)==0)
      {
          memset(tempbuf,0,512);
          memcpy(tempbuf,argv[i]+5,MIN(strlen(argv[i]) - 5,254));
          DEFAULT_LONG_SIZE = atoi(tempbuf);
          if (DEFAULT_LONG_SIZE < 100) DEFAULT_LONG_SIZE = 100;
          if (DEFAULT_LONG_SIZE > 32767) DEFAULT_LONG_SIZE = 32767;
      }
      else if (STRNCASECMP("array=",argv[i],6)==0)
      {
          memset(tempbuf,0,512);
          memcpy(tempbuf,argv[i]+6,MIN(strlen(argv[i]) - 6,254));
          DEFAULT_ARRAY_SIZE = atoi(tempbuf);
          if (DEFAULT_ARRAY_SIZE < 5) DEFAULT_ARRAY_SIZE = 5;
          if (DEFAULT_ARRAY_SIZE > 2000) DEFAULT_ARRAY_SIZE = 2000;
      }
      else if (STRNCASECMP("hash=",argv[i],5)==0)
      {
          memset(tempbuf,0,512);
          memcpy(tempbuf,argv[i]+5,MIN(strlen(argv[i]) - 5,254));
          hsize = atoi(tempbuf);
          if (hsize < 0) hsize = 0;
          if (hsize > 512) hsize = 512;
      }     
      else if (STRNCASECMP("read=",argv[i],5)==0)
      {
          memset(tempbuf,0,512);
          memcpy(tempbuf,argv[i]+5,MIN(strlen(argv[i]) - 5,254));
          bsize = atoi(tempbuf);
          if (bsize < 0) bsize = 0;
          if (bsize > 512) bsize = 512;
      }     
      else if (STRNCASECMP("batch=",argv[i],6)==0)
      {
          memset(tempbuf,0,512);
          memcpy(tempbuf,argv[i]+6,MIN(strlen(argv[i]) - 6,254));
          batch = atoi(tempbuf);
          if (batch < 0) batch = 0;
          if (batch == 1) batch = 2;
      }     
      else if (STRNCASECMP("serial=",argv[i],7)==0)
      {
          memset(tempbuf,0,512);
          memcpy(tempbuf,argv[i]+7,MIN(strlen(argv[i]) - 7,254));
          serial = atoi(tempbuf);
      }     
      else if (STRNCASECMP("trace=",argv[i],6)==0)
      {
          memset(tempbuf,0,512);
          memcpy(tempbuf,argv[i]+6,MIN(strlen(argv[i]) - 6,254));
          trace = atoi(tempbuf);
      }     
      else if (STRNCASECMP("-si",argv[i],3) == 0 && strlen(argv[i])==3)
      {
          lmode = lmode | OCI_SYSDBA;
          memset(username,0,132);
          username[0]='/';
      }
      else if (i == 1)
      {
          memset(username,0,132);
          memcpy(username,argv[i],MIN(strlen(argv[i]),131));      	
      }
    }
    
    if (strlen(sqlfname) > 0)
    {
       fp = fopen(sqlfname,"r+");
       if (fp != NULL)
       {
           while(!feof(fp))
           {
               memset(tempbuf,0,512);
               fgets(tempbuf,1023,fp);
               strcat(query,tempbuf);
               strcat(query," ");
           }
           fclose(fp);
       }
    }
    
    if (strlen(username)==0 || strlen(query)==0)
    {
       printf("Usage: %s user=... query=... field=... record=... file=...\n",argv[0]);
       printf("(@) Copyright Lou Fangxin 2004/2008, all rights reserved.\n");
       printf("Notes:\n");
       printf("       -si   = enable logon as SYSDBA\n");
       printf("       user  = username/password@tnsname\n");
       printf("       sql   = SQL file name\n");
       printf("       query = select statement\n");
       printf("       field = seperator string between fields\n");
       printf("       record= seperator string between records\n");
       printf("       head  = print row header(Yes|No)\n");
       printf("       file  = output file name(default: uldrdata.txt)\n");
       printf("       read  = set DB_FILE_MULTIBLOCK_READ_COUNT at session level\n");
       printf("       sort  = set SORT_AREA_SIZE at session level (UNIT:MB) \n");
       printf("       hash  = set HASH_AREA_SIZE at session level (UNIT:MB) \n");
       printf("       serial= set _serial_direct_read to TRUE at session level\n");
       printf("       trace = set event 10046 to given level at session level\n");
       printf("       table = table name in the sqlldr control file\n");
       printf("       mode  = sqlldr option, INSERT or APPEND or REPLACE or TRUNCATE \n");
       printf("       log   = log file name, prefix with + to append mode\n");
       printf("       long  = maximum long field size\n");
       printf("       array = array fetch size\n");
       printf("       buffer= sqlldr READSIZE and BINDSIZE, default 16 (MB)\n");
       printf("       form  = display rows as form (yes or no)\n");
       printf("\n");
       printf("  for field and record, you can use '0x' to specify hex character code,\n");
       printf("  \\r=0x%02x \\n=0x%02x |=0x%0x ,=0x%02x \\t=0x%02x\n",'\r','\n','|',',','\t');
       exit(0);
    }

    if (DEFAULT_ARRAY_SIZE * DEFAULT_LONG_SIZE > 104857600)
    {
        DEFAULT_ARRAY_SIZE = 104857600/DEFAULT_LONG_SIZE;
        if (DEFAULT_ARRAY_SIZE < 5) DEFAULT_ARRAY_SIZE=5;
    }

    if (strlen(logfile))
    {
       fp_log = openFile(logfile,tempbuf,0);
    }

    if(LogonDB(&conn,username,NULL,lmode))
    {
       if (fp_log != NULL) fclose(fp_log);
       exit(1);
    }

    if(CreateSQL(&conn,&stmt) == 0)
    { 

        ParseSQL(&conn,&stmt,"ALTER SESSION SET NLS_DATE_FORMAT='YYYY-MM-DD HH24:MI:SS'");
        executeStatement(&conn,&stmt);
        ParseSQL(&conn,&stmt,"ALTER SESSION SET NLS_TIMESTAMP_FORMAT='YYYY-MM-DD HH24:MI:SSXFF'");
        executeStatement(&conn,&stmt);
        ParseSQL(&conn,&stmt,"ALTER SESSION SET NLS_TIMESTAMP_TZ_FORMAT='YYYY-MM-DD HH24:MI:SSXFF TZH:TZM'");
        executeStatement(&conn,&stmt);

        if (bsize)
        {
	   memset(tempbuf,0,512);
           sprintf(tempbuf,"ALTER SESSION SET DB_FILE_MULTIBLOCK_READ_COUNT=%d",bsize);
           ParseSQL(&conn,&stmt,tempbuf);
           executeStatement(&conn,&stmt);
        }
        if (hsize)
        {
	   memset(tempbuf,0,512);
           sprintf(tempbuf,"ALTER SESSION SET HASH_AREA_SIZE=%d",hsize * 1048576);
           ParseSQL(&conn,&stmt,tempbuf);
           executeStatement(&conn,&stmt);
	   memset(tempbuf,0,512);
           sprintf(tempbuf,"ALTER SESSION SET \"_hash_multiblock_io_count\"=128");
           ParseSQL(&conn,&stmt,tempbuf);
           executeStatement(&conn,&stmt);
        }
        if (serial)
        {
	   memset(tempbuf,0,512);
           sprintf(tempbuf,"ALTER SESSION SET \"_serial_direct_read\"=TRUE");
           ParseSQL(&conn,&stmt,tempbuf);
           executeStatement(&conn,&stmt);
        }
        if (ssize)
        {
	   memset(tempbuf,0,512);
           sprintf(tempbuf,"ALTER SESSION SET SORT_AREA_SIZE=%d",ssize * 1048576);
           ParseSQL(&conn,&stmt,tempbuf);
           executeStatement(&conn,&stmt);
	   memset(tempbuf,0,512);
           sprintf(tempbuf,"ALTER SESSION SET SORT_AREA_RETAINED_SIZE=%d",ssize * 1048576);
           ParseSQL(&conn,&stmt,tempbuf);
           executeStatement(&conn,&stmt);
	   memset(tempbuf,0,512);
           sprintf(tempbuf,"ALTER SESSION SET \"_sort_multiblock_read_count\"=128");
           ParseSQL(&conn,&stmt,tempbuf);
           executeStatement(&conn,&stmt);
        }
        if (trace)
        {
	   memset(tempbuf,0,512);
           sprintf(tempbuf,"ALTER SESSION SET EVENTS='10046 TRACE NAME CONTEXT FOREVER,LEVEL %d'", trace);
           ParseSQL(&conn,&stmt,tempbuf);
           executeStatement(&conn,&stmt);
        }
        if(ParseSQL(&conn,&stmt,query) == 0)
        {
            /*
               Pass parameter value here
            */
            for(i=0;i<argc;i++)
            {
               if (STRNCASECMP("arg:",argv[i],4) == 0)
               {        
                  memset(argname[argcount],0,128);
                  memset(argval[argcount] ,0,128);
                  memset(tempbuf,0,512);
                  memcpy(tempbuf,argv[i]+4,MIN(strlen(argv[i]) - 4,254));
                  for(n=0;n<strlen(tempbuf);n++) if (tempbuf[n]=='=') break;
                  memcpy(argname[argcount], tempbuf, n);
                  if(n+1<strlen(tempbuf)) memcpy(argval[argcount], tempbuf+n+1, strlen(tempbuf) - n - 1);
                  if(strlen(argname[argcount])>0)
		  {
			BindValue(&conn,&stmt,argname[argcount],argval[argcount]);
			argcount ++;
		  }
		  if (argcount >= 20) break;
               }
            }

            if(executeStatement(&conn,&stmt) == 0)
            {
                if(strlen(tabname))
                {
                    memset(ctlfname,0,256);
                    sprintf(ctlfname,"%s_sqlldr.ctl",tabname);
                    fpctl = fopen(ctlfname,"wb+");
                    
                    if(fpctl != NULL)
                    {
                       fprintf(fpctl,"--\n");
                       fprintf(fpctl,"-- Generated by OCIULDR\n");
                       fprintf(fpctl,"--\n");
                       if (!header)
                          fprintf(fpctl,"OPTIONS(BINDSIZE=%d,READSIZE=%d,ERRORS=-1,ROWS=50000)\n", buffer, buffer);
                       else
                          fprintf(fpctl,"OPTIONS(BINDSIZE=%d,READSIZE=%d,SKIP=1,ERRORS=-1,ROWS=50000)\n", buffer, buffer);
                       fprintf(fpctl,"LOAD DATA\n");
                       fprintf(fpctl,"INFILE '%s' \"STR X'", fname);
                       for(i=0;i<strlen(record);i++) fprintf(fpctl,"%02x",record[i]);
                       fprintf(fpctl,"'\"\n");
                       fprintf(fpctl,"%s INTO TABLE %s\n", tabmode, tabname);
                       fprintf(fpctl,"FIELDS TERMINATED BY X'");
                       for(i=0;i<strlen(field);i++) fprintf(fpctl,"%02x",field[i]);
                       fprintf(fpctl,"' TRAILING NULLCOLS \n");
                    }
                }
                if(getColumns(fpctl,&conn,&stmt,&col, (field[0]==0x20?1:0)) > 0)
                {
                      /*
                      iobuf = (text *)malloc(DEFAULT_BUFFER_SIZE);
                      memset(iobuf,0,DEFAULT_BUFFER_SIZE);
                      setvbuf(fp,iobuf,_IOFBF,DEFAULT_BUFFER_SIZE);
                      printRow(fp,&conn,&stmt,&col,field,flen,record,rlen);
                      fclose(fp);
                      free(iobuf);
                      */
                      printRow(fname,&conn,&stmt,&col,field,flen,record,rlen,batch, header, frmdisp, (field[0]==0x20?1:0));
                }
                else
                {
                      return_code = 5;
                }
                if (fpctl != NULL) fclose(fpctl);
             }
             else
             {
                return_code = 4;
             }
        }
        else
        {
            return_code = 3;
        }
      	oclose(&stmt);
    } 
    else
    {
	return_code = 2;
    }

    freeColumn(&col);
    ologof(&conn);
    if (fp_log != NULL) fclose(fp_log);
    
    return return_code;
}
```

# tbuldr.c

```c
* 
* tbuldr [ToolBox*UnLoader]
* Author: NinGoo [http://www.ningoo.net]
* Rewrite with OCI8 base on OCIULDR by Lou Fangxin [http://www.anysql.net]
* change history:
*  2008-04-16 First issued. version 2.20
*  2008-04-17 Fix some bugs on windows;add display elapsed time on log. version 2.21
*/
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <oci.h>
#include <oratypes.h>
#include <ocidfn.h>
#ifdef __STDC__
#include <ociapr.h>
#else
#include <ocikpr.h>
#endif
#include <ocidem.h>

#if defined(_WIN32)
#define STRNCASECMP memicmp
#else
#define STRNCASECMP strncasecmp
#endif

#define  MIN(a,b) ((a) > (b) ? (b) : (a))

/* constants */
#define MAJOR_VERSION_NUMBER       2
#define MINOR_VERSION_NUMBER       21
#define MAX_SELECT_LIST_SIZE       1024
#define MAX_COLNAME_BUFFER_SIZE    32
#define MAX_CLOB_SIZE              65534 
#define MAX_IO_BUFFER_SIZE         1048576      

/* return code */
#define SUCCESSFUL        0
#define ERROR_LOGON       1
#define ERROR_PREPARE_SQL 2
#define ERROR_EXECUTE_SQL 3
#define ERROR_METADATA    4
#define ERROR_OUT_FILE    5
#define ERROR_FETCH       6
#define ERROR_OTHER       100

/*global env variables*/
static OCIEnv       *p_env  = (OCIEnv *)0;
static OCIServer    *p_srv  = (OCIServer *)0;
static OCIError     *p_err  = (OCIError *)0;
static OCISvcCtx    *p_svc  = (OCISvcCtx *)0;
static OCIStmt      *p_stmt = (OCIStmt *)0;
static OCIDescribe  *p_desc = (OCIDescribe *)0;
static OCISession   *p_sess = (OCISession *)0;

struct COLUMN
{
  /* Describe */
  ub2             colwidth;
  ub2             coltype;
  ub2             precision;
  ub2             scale;
  text            fmtstr[64]; // for fixlen output
  text            *colname;

  /*+ Fetch */
  OCIDefine       *p_define;
  ub1             *colbuf;    //output variable
  sb2             *p_indv;
  ub2             *col_retlen;
  ub2             *col_retcode;

  /*+ Point to next column */
  struct          COLUMN *next;
};

struct PARAM
{
  text query[1024];
  text field[32];
  text record[32];
  text enclose[32];
  text connstr[132];
  text fname[128];
  text sqlfname[128];
  text logfile[128];
  text tabname[128];
  text tabmode[16];

  ub1  field_len;   // field length
  ub1  return_len;  // return length
  ub1  enclose_len; // enclose length
  ub4  buffer;      // buffer size,default 16MB
  ub2  hsize;       // hash_area_size
  ub2  ssize;       // sort_area_size
  ub2  bsize;       // db_file_multiblock_read_count
  ub1  serial;      // _serial_direct_read
  ub1  trace;       // 10046 trace level
  ub2  batch;       // batch out files
  ub1  header;      // output header line
  ub4  feedback;    // display progress log every x rows
  
  ub4  asize;       // array size
  ub4  lsize;       // long size 

  ub1 isSTDOUT;     // stdout, not file
  ub1 isForm;       // display as MySQL form format
  ub1 isFixlen;     // fixed length output format
  ub1 debug;        // debug mode,more log output
};

struct COLUMN col;
struct PARAM *param;
int    return_code = SUCCESSFUL;
int    row_totallen=0;
FILE   *fp_log = NULL;
FILE   *fp_ctl = NULL;

/* functions */
void env_init();
void env_exit();

void session_init();

sword db_logon(char *v_user,char *v_pass,char *v_host);
void  db_logout();

void decode_connstr(char *v_user,char *v_pass,char *v_host);
void check_err(OCIError *p_err,sword status);
void print_help();

sword sql_prepare(OCIStmt *p_stmt, text *sql_statement);
sword sql_execute(OCISvcCtx *p_svc,OCIStmt *p_stmt,ub4 execount);

sword get_columns(OCIStmt *p_stmt, struct COLUMN *collist);
void  free_columns(struct COLUMN *col);

void print_feedback(ub4 row);
void print_row(OCISvcCtx *p_svc,OCIStmt *p_stmt,struct COLUMN *col);

int  get_param(int argc,char **agrv);
void sqlldr_ctlfile(struct COLUMN *collist,ub2 numcols);

int convert_option(const ub1 *src, ub1* dst, int mlen);
ub1 get_hexindex(char c);

FILE *open_file(const text *fname, text tempbuf[], int batch);

int main(int argc, char *argv[])
{
  char *p_user=malloc(32);
  char *p_pass=malloc(32);
  char *p_host=malloc(32);

  if(get_param(argc,argv)) return return_code;
  
  if(strlen(param->connstr)){
    decode_connstr(p_user,p_pass,p_host);
  }
  else{
    return_code=ERROR_OTHER;
    print_help();
  }
  
  env_init();

  if(db_logon(p_user,p_pass,p_host)==1)
  {
    return_code=ERROR_LOGON;
    return return_code;
  }

  session_init();

  if(sql_prepare(p_stmt,param->query)==0)
  {
    if(sql_execute(p_svc,p_stmt,0)==0)
    {
      get_columns(p_stmt,&col);
      print_row(p_svc,p_stmt,&col);
      free_columns(&col);
    }
  }

  db_logout();
  env_exit();

  return return_code;
}

/* ----------------------------------------------------------------- */
/* initialize environment, allocate handles, etc.                    */
/* ----------------------------------------------------------------- */
void env_init ()
{
  OCIEnvCreate((OCIEnv **) &p_env,OCI_THREADED|OCI_OBJECT,(dvoid *)0,
        (dvoid * (*)(dvoid *, size_t)) 0,
        (dvoid * (*)(dvoid *, dvoid *, size_t))0,
        (void (*)(dvoid *, dvoid *)) 0,
        (size_t) 0, (dvoid **) 0);

  /* error handle */
  OCIHandleAlloc ((dvoid *) p_env, (dvoid **) &p_err, OCI_HTYPE_ERROR,
                         (size_t) 0, (dvoid **) 0);

  /* server handle */
  OCIHandleAlloc ((dvoid *) p_env, (dvoid **) &p_srv, OCI_HTYPE_SERVER,
                         (size_t) 0, (dvoid **) 0);
  /* svcctx handle*/
  OCIHandleAlloc ((dvoid *) p_env, (dvoid **) &p_svc, OCI_HTYPE_SVCCTX,
                         (size_t) 0, (dvoid **) 0);

  /* set attribute server context in the service context */
  OCIAttrSet ((dvoid *) p_svc, OCI_HTYPE_SVCCTX, (dvoid *)p_srv,
                     (ub4) 0, OCI_ATTR_SERVER, (OCIError *) p_err);

  /*stmt handle*/
  OCIHandleAlloc((dvoid *) p_env, (dvoid **) &p_stmt, OCI_HTYPE_STMT,
                 (size_t) 0, (dvoid **) 0);
}

/* ----------------------------------------------------------------- */
/* attach to the server and log on                                   */
/* ----------------------------------------------------------------- */
sword db_logon (char *v_user,char *v_pass,char *v_host)
{
  sword rc;

  OCIHandleAlloc ((dvoid *) p_env, (dvoid **)&p_sess,
                  (ub4) OCI_HTYPE_SESSION, (size_t) 0, (dvoid **) 0);

  if (strlen(v_host)==0)
  {
    check_err(p_err,OCIServerAttach (p_srv,p_err,(text *)0,(sb4)0,OCI_DEFAULT));
  }		
  else
  {
    check_err(p_err,OCIServerAttach (p_srv,p_err,(text *)v_host,(sb4)strlen("v_host"),OCI_DEFAULT));
  } 

  OCIAttrSet ((dvoid *)p_sess, (ub4)OCI_HTYPE_SESSION,
              (dvoid *)v_user, (ub4)strlen((char *)v_user),
              OCI_ATTR_USERNAME, p_err);

  OCIAttrSet ((dvoid *)p_sess, (ub4)OCI_HTYPE_SESSION,
              (dvoid *)v_pass, (ub4)strlen((char *)v_pass),
              OCI_ATTR_PASSWORD, p_err);

  rc=OCISessionBegin(p_svc, p_err, p_sess, OCI_CRED_RDBMS,OCI_DEFAULT);

  if(rc)
  {
    check_err(p_err, rc);
    return 1;
  }

  OCIAttrSet ((dvoid *) p_svc, (ub4) OCI_HTYPE_SVCCTX,
              (dvoid *) p_sess, (ub4) 0,
              (ub4) OCI_ATTR_SESSION, p_err);

  if(v_user) free(v_user);
  if(v_pass) free(v_pass);
  if(v_host) free(v_host);

  return 0;
}

/*-------------------------------------------------------------------*/
/* Logoff and disconnect from the server.                            */
/*-------------------------------------------------------------------*/
void db_logout()
{
  if (param->trace)
  {
    sql_prepare(p_stmt,"ALTER SESSION SET EVENTS='10046 TRACE NAME CONTEXT OFF'");
    sql_execute(p_svc,p_stmt,1);
  }

  OCIHandleFree ((dvoid *) p_stmt, (ub4) OCI_HTYPE_STMT);

  OCISessionEnd (p_svc, p_err, p_sess, (ub4) 0);
}

/*-------------------------------------------------------------------*/
/* Free handles and exit.                                            */
/*-------------------------------------------------------------------*/
void env_exit()
{
  if (p_err)  OCIServerDetach (p_srv, p_err, (ub4) OCI_DEFAULT );
  if (p_srv)  OCIHandleFree((dvoid *) p_srv, (CONST ub4) OCI_HTYPE_SERVER);
  if (p_svc)  OCIHandleFree((dvoid *) p_svc, (CONST ub4) OCI_HTYPE_SVCCTX);
  if (p_err)  OCIHandleFree((dvoid *) p_err, (CONST ub4) OCI_HTYPE_ERROR);
  if (p_sess) OCIHandleFree((dvoid *) p_sess,(CONST ub4) OCI_HTYPE_SESSION);
  if (p_stmt) OCIHandleFree((dvoid *) p_stmt,(CONST ub4) OCI_HTYPE_STMT);

  if (fp_ctl) fclose(fp_ctl);
  if (fp_log) fclose(fp_log);

  if (param) free(param);

//  exit(return_code);
}

/* ----------------------------------------------------------------- */
/* retrieve error message and print it out.                          */
/* ----------------------------------------------------------------- */
void check_err(OCIError *p_err,sword status)
{
  text errbuf[512];
  sb4 errcode = 0;

  switch (status)
  {
  case OCI_SUCCESS:
    break;
  case OCI_SUCCESS_WITH_INFO:
    (void) printf("Error - OCI_SUCCESS_WITH_INFO\n");
    (void) OCIErrorGet ((dvoid *)p_err, (ub4) 1, (text *) NULL, &errcode,
                    errbuf, (ub4) sizeof(errbuf), OCI_HTYPE_ERROR);
    (void) printf("        %.*s\n", 512, errbuf);
    break;
  case OCI_NEED_DATA:
    (void) printf("Error - OCI_NEED_DATA\n");
    break;
  case OCI_NO_DATA:
    (void) printf("Error - OCI_NODATA\n");
    break;
  case OCI_ERROR:
    (void) OCIErrorGet ((dvoid *)p_err, (ub4) 1, (text *) NULL, &errcode,
                    errbuf, (ub4) sizeof(errbuf), OCI_HTYPE_ERROR);
    (void) printf("Error - %.*s\n", 512, errbuf); 
    break;
  case OCI_INVALID_HANDLE:
    (void) printf("Error - OCI_INVALID_HANDLE\n");
    break;
  case OCI_STILL_EXECUTING:
    (void) printf("Error - OCI_STILL_EXECUTE\n");
    break;
  case OCI_CONTINUE:
    (void) printf("Error - OCI_CONTINUE\n");
    break;
  default:
    break;
  }
}

/* ----------------------------------------------------------------- */
/* get username,password,alias from use= string                      */
/* ----------------------------------------------------------------- */
void decode_connstr(char *v_user,char *v_pass,char *v_host)
{
  char *r1="/";
  char *r2="@";
  int  n1=0;
  int  n2=0;

  n1=strcspn(param->connstr,r1);
  n2=strcspn(param->connstr,r2);

  strncpy(v_user,param->connstr,n1);

  if( n2>n1 ){
    strncpy(v_pass,&param->connstr[n1+1],n2-n1-1);
  }

  if( strlen(param->connstr)>n2){
    strncpy(v_host,&param->connstr[n2+1],strlen(param->connstr)-n2-1);
  }
}

/* ----------------------------------------------------------------- */
/* print help information                                            */
/* ----------------------------------------------------------------- */
void print_help()
{
  printf("\n----------------------------------------------------------------------------\n");
  printf("- tbuldr: Release %d.%d (ociuldr)\n",MAJOR_VERSION_NUMBER,MINOR_VERSION_NUMBER);
  printf("- First issued by Lou Fangxin (@) Copyright 2004/2008, all rights reserved.\n");
  printf("- Modified by Ning Haiyuan (http://www.NinGoo.net).\n");
  printf("----------------------------------------------------------------------------\n");
  printf(" Usage: TBULDR keyword=value [,keyword=value,...]\n");
  printf(" Valid Keywords:\n");
  printf("       user     = username/password@tnsname\n");
  printf("       query    = select statement, can simply speicify a table name\n");
  printf("       sql      = SQL file name\n");
  printf("       field    = seperator string between fields\n");
  printf("       record   = seperator string between records\n");
  printf("       enclose  = fields enclose string\n");
  printf("       file     = output file name, default: uldrdata.txt\n");
  printf("       head     = print row header(Yes|No,ON|OFF,1|0)\n");
  printf("       read     = set DB_FILE_MULTIBLOCK_READ_COUNT at session level\n");
  printf("       sort     = set SORT_AREA_SIZE at session level (UNIT:MB) \n");
  printf("       hash     = set HASH_AREA_SIZE at session level (UNIT:MB) \n");
  printf("       serial   = set _serial_direct_read to TRUE if 1 at session level\n");
  printf("       trace    = set event 10046 to given level at session level\n");
  printf("       table    = table name in the sqlldr control file\n");
  printf("       mode     = sqlldr option, INSERT or APPEND or REPLACE or TRUNCATE \n");
  printf("       log      = log file name, prefix with + to append mode\n");
  printf("       long     = maximum long field size, default 8192 max 65534\n");
  printf("       array    = array fetch size, default 50\n");
  printf("       buffer   = sqlldr READSIZE and BINDSIZE, default 16 (MB)\n");
  printf("       feedback = display progress every x rows, default 500000\n");
  printf("       form     = display rows as form (Yes|No)\n");
  printf("       fixlen   = fix length format (Yes|No)\n");
  printf("\n");
  printf("  for field, record and enclose, use '0x' to specify hex character code\n");
  printf("  \\r=0x%02x \\n=0x%02x |=0x%0x ,=0x%02x \\t=0x%02x",'\r','\n','|',',','\t');
  printf("  (more: man ascii)\n\n");

  return_code=ERROR_OTHER;
}

/* ----------------------------------------------------------------- */
/* prepare a sql statement                                           */
/* ----------------------------------------------------------------- */
sword sql_prepare(OCIStmt *p_stmt, text *sql_statement)
{
  sword rc;
  rc=OCIStmtPrepare(p_stmt, p_err, (text *) sql_statement,
                      (ub4) strlen(sql_statement), OCI_NTV_SYNTAX, OCI_DEFAULT);

  if (rc!=0)
  {
    check_err(p_err,rc);
    return_code=ERROR_LOGON;
    return -1;
  }
  else
    return 0;
}

/* ----------------------------------------------------------------- */
/* execute a sql statement                                           */
/* ----------------------------------------------------------------- */
sword sql_execute(OCISvcCtx *p_svc,OCIStmt *p_stmt,ub4 execount)
{
  sword rc;
  rc=OCIStmtExecute(p_svc,p_stmt,p_err,execount,0,NULL,NULL,OCI_DEFAULT);

  if (rc!=0)
  {
   check_err(p_err,rc);
   return -1;
  }
  return 0;
}

/* ----------------------------------------------------------------- */
/* initialize session, set date format                               */
/* ----------------------------------------------------------------- */
void session_init()
{
  text tempbuf[1024];

  sql_prepare(p_stmt,"ALTER SESSION SET NLS_DATE_FORMAT='YYYY-MM-DD HH24:MI:SS'");
  sql_execute(p_svc,p_stmt,1);

  sql_prepare(p_stmt,"ALTER SESSION SET NLS_TIMESTAMP_FORMAT='YYYY-MM-DD HH24:MI:SSXFF'");
  sql_execute(p_svc,p_stmt,1);

  sql_prepare(p_stmt,"ALTER SESSION SET NLS_TIMESTAMP_TZ_FORMAT='YYYY-MM-DD HH24:MI:SSXFF TZH:TZM'");
  sql_execute(p_svc,p_stmt,1);

  if(param->bsize)
  {
    memset(tempbuf,0,1024);
    sprintf(tempbuf,"ALTER SESSION SET DB_FILE_MULTIBLOCK_READ_COUNT=%d",param->bsize);
    sql_prepare(p_stmt,tempbuf);
    sql_execute(p_svc,p_stmt,1);
  }

  if(param->hsize)
  {
    memset(tempbuf,0,1024);
    sprintf(tempbuf,"ALTER SESSION SET HASH_AREA_SIZE=%d",param->hsize*1048576);
    sql_prepare(p_stmt,tempbuf);
    sql_execute(p_svc,p_stmt,1);
  }

  if(param->ssize)
  {
    memset(tempbuf,0,1024);
    sprintf(tempbuf,"ALTER SESSION SET SORT_AREA_SIZE=%d",param->ssize*1048576);
    sql_prepare(p_stmt,tempbuf);
    sql_execute(p_svc,p_stmt,1);

    memset(tempbuf,0,1024);
    sprintf(tempbuf,"ALTER SESSION SET SORT_AREA_RETAINED_SIZE=%d",param->ssize*1048576);
    sql_prepare(p_stmt,tempbuf);
    sql_execute(p_svc,p_stmt,1);

    sql_prepare(p_stmt,"ALTER SESSION SET \"_sort_multiblock_read_count\"=128");
    sql_execute(p_svc,p_stmt,1);
  }
  
  if (param->serial)
  {
    memset(tempbuf,0,1024);
    sprintf(tempbuf,"ALTER SESSION SET \"_serial_direct_read\"=TRUE");
    sql_prepare(p_stmt,tempbuf);
    sql_execute(p_svc,p_stmt,1);
  }

  if (param->trace)
  {
    memset(tempbuf,0,1024);
    sprintf(tempbuf,"ALTER SESSION SET EVENTS='10046 TRACE NAME CONTEXT FOREVER,LEVEL %d'", param->trace);
    sql_prepare(p_stmt,tempbuf);
    sql_execute(p_svc,p_stmt,1);
  }
}

/* ----------------------------------------------------------------- */
/* get column definition                                             */
/* ----------------------------------------------------------------- */
sword get_columns(OCIStmt *p_stmt, struct COLUMN *collist)
{
  OCIParam      *p_param = (OCIParam *) 0;
  ub4           col;
  ub2           numcols;       //select-list columns
  struct COLUMN *tempcol;
  struct COLUMN *nextcol;

  nextcol = collist;

  //get table describ info
  if(!param->isSTDOUT) printf("\n");

  //get columns
  check_err(p_err,OCIAttrGet(p_stmt, OCI_HTYPE_STMT, &numcols,0, OCI_ATTR_PARAM_COUNT, p_err));

  /* Describe the select-list items. */
  for (col = 0; col < numcols; col++)
  {
    tempcol = (struct COLUMN *) malloc(sizeof(struct COLUMN));
    tempcol-> p_indv        = (sb2 *)malloc(param->asize * sizeof(sb2));
    tempcol-> col_retlen  = (ub2 *)malloc(param->asize * sizeof(ub2));
    tempcol-> col_retcode = (ub2 *)malloc(param->asize * sizeof(ub2));
    tempcol-> colname     = malloc(MAX_COLNAME_BUFFER_SIZE);
    memset(tempcol-> colname,0,MAX_COLNAME_BUFFER_SIZE);
    memset(tempcol->p_indv,0,param->asize * sizeof(sb2));
    memset(tempcol->col_retlen,0,param->asize * sizeof(ub2));
    memset(tempcol->col_retcode,0,param->asize * sizeof(ub2));

    tempcol->next = NULL;
    tempcol->colbuf = NULL;
    memset(tempcol->fmtstr,0,64);

    /* get parameter for column col*/
    check_err(p_err, OCIParamGet(p_stmt, OCI_HTYPE_STMT, p_err, (void **)&p_param, col+1));
    /* get data-type of column col */
    check_err(p_err, OCIAttrGet(p_param, OCI_DTYPE_PARAM,&tempcol->coltype, 0, OCI_ATTR_DATA_TYPE, p_err));
    check_err(p_err, OCIAttrGet(p_param, OCI_DTYPE_PARAM,&tempcol->colname, 0/*&tempcol->colname_len*/, (ub4)OCI_ATTR_NAME, p_err));
    check_err(p_err, OCIAttrGet(p_param, OCI_DTYPE_PARAM,&tempcol->colwidth, 0, OCI_ATTR_DATA_SIZE, p_err));
    check_err(p_err, OCIAttrGet(p_param, OCI_DTYPE_PARAM,&tempcol->precision, 0, OCI_ATTR_PRECISION, p_err));
    check_err(p_err, OCIAttrGet(p_param, OCI_DTYPE_PARAM,&tempcol->scale, 0, OCI_ATTR_SCALE,p_err));

    nextcol->next = tempcol;
    nextcol=tempcol;

    switch(nextcol->coltype)
    {
      case SQLT_DATE:
      case SQLT_DAT:
        nextcol->colwidth=19;
//        nextcol->coltype=(param->isFixlen?SQLT_AVC:STRING_TYPE);
        break;
      case SQLT_TIMESTAMP: /* TIMESTAMP */
        nextcol->colwidth=26;
        break;
      case SQLT_TIMESTAMP_TZ: /* TIMESTAMP WITH TIMEZONE */
        nextcol->colwidth=33;
//        nextcol->coltype=(param->isFixlen?SQLT_AVC:STRING_TYPE);
        break;
      case SQLT_LBI:  /* LONG RAW */
      case SQLT_BLOB:
        nextcol->colwidth=param->lsize;
//        nextcol->coltype=(param->isFixlen?SQLT_AVC:STRING_TYPE);
        break;
      case SQLT_CLOB:  
        nextcol->colwidth=param->lsize;
//        nextcol->coltype=(param->isFixlen?SQLT_AVC:STRING_TYPE);
        break;
      case SQLT_RDD:
        nextcol->colwidth=18;
//        nextcol->coltype=(param->isFixlen?SQLT_AVC:STRING_TYPE);
        break;
      case SQLT_INT:
      case SQLT_NUM:
        if (nextcol->precision)
          nextcol->colwidth=nextcol->precision;
        else
          nextcol->colwidth=38;
//        nextcol->coltype=(param->isFixlen?SQLT_AVC:STRING_TYPE);
        break;
      case SQLT_FILE: /* BFILE */
        nextcol->colwidth=param->lsize;
 //       nextcol->coltype=(param->isFixlen?SQLT_AVC:STRING_TYPE);
        break;
      default:
        nextcol->coltype=(param->isFixlen?SQLT_AVC:STRING_TYPE);
        break;
      }

      if (nextcol->colwidth > param->lsize || nextcol->colwidth == 0)
      {
         nextcol->colwidth = param->lsize;
      }

      if(param->enclose_len) tempcol->colwidth=tempcol->colwidth+2*param->enclose_len;

      row_totallen = row_totallen + nextcol->colwidth;
      sprintf(nextcol->fmtstr,"%%-%ds", nextcol->colwidth);

      /* add some more byte to buffer */
      nextcol->colwidth++;
      nextcol->colbuf = malloc((int)param->asize * nextcol->colwidth);
      memset(nextcol->colbuf,0,((int)param->asize * nextcol->colwidth));
      
      if(param->debug)
         printf("%10d bytes allocated for column %s (%d) type %d\n",
                nextcol->colwidth*param->asize,nextcol->colname,nextcol->colwidth-1,nextcol->coltype);

      //define output
      if (nextcol->coltype==SQLT_BLOB || nextcol->coltype==SQLT_LBI )
      {
        check_err(p_err,OCIDefineByPos(p_stmt, &nextcol->p_define, p_err, col+1,
                       (dvoid *) nextcol->colbuf,nextcol->colwidth, SQLT_LBI, nextcol->p_indv,
                       (ub2 *)nextcol->col_retlen,(ub2 *)nextcol->col_retcode, OCI_DEFAULT));	
      }
      else
      {
        check_err(p_err,OCIDefineByPos(p_stmt, &nextcol->p_define, p_err, col+1, 
                       (dvoid *) nextcol->colbuf,nextcol->colwidth, SQLT_STR, nextcol->p_indv,
                       (ub2 *)nextcol->col_retlen,(ub2 *)nextcol->col_retcode, OCI_DEFAULT));
      }	
  }
  
  sqlldr_ctlfile(collist,numcols);

  return 0;
}

/* ----------------------------------------------------------------- */
/* free column datastruct                                            */
/* ----------------------------------------------------------------- */
void free_columns(struct COLUMN *col)
{
   struct COLUMN *p,*temp;
   p=col->next;

   col->next = NULL;
   while(p!=NULL)
   {
     free(p->colbuf);
     free(p->p_indv);
     free(p->col_retlen);
     free(p->col_retcode);

     temp=p;
     p=temp->next;
     free(temp);
   }
}

/* ----------------------------------------------------------------- */
/* output query result                                               */
/* ----------------------------------------------------------------- */
void print_row(OCISvcCtx *p_svc,OCIStmt *p_stmt,struct COLUMN *col)
{
  ub4    colcount;
  ub4    tmp_rows;
  ub4    tmp_rows_size;
  ub4    rows;
  ub4    j;
  ub4    seconds;
  sword  rc;
  sword  r;
  ub4    c;
  ub4    trows;
  struct COLUMN *p;
  struct COLUMN *cols[MAX_SELECT_LIST_SIZE];
  text   tempbuf[512];
  FILE   *fp = NULL;
  FILE   *fp_lob = NULL;
  ub1    *iobuf = NULL;

  time_t start_time,end_time;
  
  int bcount=1;
  trows=0;
  colcount=0;

  start_time=time(0);
  
  p = col->next;
  while(p != NULL)
  {
    cols[colcount] = p;
    p=p->next;
    colcount++;
  }
  
  memset(tempbuf,0,512);

  if(!param->isSTDOUT)
  {
    if((fp = open_file(param->fname,tempbuf,bcount)) == NULL) 
    {
      fprintf((fp_log == NULL?stderr:fp_log),"ERROR -- Cannot write to file : %s\n", tempbuf);
      return_code=ERROR_OUT_FILE;
      return;
    }

    iobuf = (text *)malloc(MAX_IO_BUFFER_SIZE);
    memset(iobuf,0,MAX_IO_BUFFER_SIZE);
    if(setvbuf(fp,iobuf,_IOFBF,MAX_IO_BUFFER_SIZE))
      printf("Canot setup buffer for output file!\n");
  }

  if (param->header)
  {
    for(c=0;c<colcount;c++)
    {
      if (param->isFixlen)
        fprintf(fp,cols[c]->fmtstr,cols[c]->colname);
      else
      {
        fprintf((fp == NULL?stdout:fp),"%s",cols[c]->colname);
        if (c < colcount - 1) 
          fwrite(param->field,param->field_len,1,(fp == NULL?stdout:fp));
      }
    }
    if(!param->isFixlen) fwrite(param->record,param->return_len,1,(fp == NULL?stdout:fp));
  }

  if(param->debug) fprintf((fp_log == NULL?stderr:fp_log),"\n");
  print_feedback(trows);
  for (;;)
  {
    rows = param->asize;
    rc=OCIStmtFetch(p_stmt, p_err, param->asize, 0, OCI_DEFAULT);    

    if (rc !=0)
    {
      if (rc!= OCI_NO_DATA) check_err(p_err,rc);
      
      check_err(p_err,OCIAttrGet((dvoid *) p_stmt, (ub4) OCI_HTYPE_STMT,(dvoid *)&tmp_rows,
                 (ub4 *) &tmp_rows_size, (ub4)OCI_ATTR_ROWS_FETCHED,p_err));
      rows = tmp_rows % param->asize;
    }		
    
    for(r=0;r<rows;r++)
    {
      for(c=0;c<colcount;c++)
      {
        if (param->isForm)
        {
          fprintf((fp == NULL?stdout:fp),"%-31s: ",cols[c]->colname);
        }
        if (*(cols[c]->p_indv+r) >= 0)
        {
          if (cols[c]->coltype == SQLT_LBI || cols[c]->coltype == SQLT_BLOB) //long binary type
          {  
            if(param->enclose_len) fwrite(param->enclose,param->enclose_len,1,(fp == NULL?stdout:fp));
            for(j=0;j < *(cols[c]->col_retlen+r); j++)
            {
               fprintf((fp == NULL?stdout:fp), "%02x", cols[c]->colbuf[r * cols[c]->colwidth + j]);
            }
            if(param->enclose_len) fwrite(param->enclose,param->enclose_len,1,(fp == NULL?stdout:fp));
          }
          else
          { 
            if(param->isFixlen)
              fprintf(fp,cols[c]->fmtstr, cols[c]->colbuf+(r* cols[c]->colwidth));
            else
            {
              if(param->enclose_len) fwrite(param->enclose,param->enclose_len,1,(fp == NULL?stdout:fp));
/*
              ub1 *p_tmp=cols[c]->colbuf+(r* cols[c]->colwidth);
              while(*p_tmp)
              {
                if(STRNCASECMP(p_tmp,param->field,param->field_len)==0) memcpy(p_tmp," ",param->field_len);
                p_tmp=p_tmp+param->field_len;
              }
*/
              fwrite(cols[c]->colbuf+(r* cols[c]->colwidth),*(cols[c]->col_retlen+r),1,(fp == NULL?stdout:fp));
              if(param->enclose_len) fwrite(param->enclose,param->enclose_len,1,(fp == NULL?stdout:fp));
            }
          }
        }
        if (param->isForm)
        {
          fprintf((fp == NULL?stdout:fp),"\n");
        }
        else
        {
          if (c < colcount - 1)
            if(!param->isFixlen) fwrite(param->field,param->field_len,1,(fp == NULL?stdout:fp));
        }  
      }
      if(!param->isFixlen) fwrite(param->record,param->return_len,1,(fp == NULL?stdout:fp));
      trows=trows+1;
      if (trows % param->feedback  == 0)
      {
        print_feedback(trows);
        if(param->batch && ((trows / param->feedback) % param->batch) == 0)
        {
          if(!param->isSTDOUT)
            fprintf((fp_log == NULL?stderr:fp_log),"         output file %s closed at %u rows.\n", tempbuf, trows);
          if(fp) fclose(fp);
          bcount++;
          memset(tempbuf,0,512);

          if(param->isSTDOUT)
          {
            if((fp = open_file(param->fname,tempbuf,bcount)) == NULL) 
            {
              fprintf((fp_log == NULL?stderr:fp_log),"ERROR -- Cannot write to file : %s\n", tempbuf);
              return_code=ERROR_OUT_FILE;
              return;
            }
          }

          if (param->header)
          {
            for(c=0;c<colcount;c++)
            {
              fprintf(fp,"%s",cols[c]->colname);
              if (c < colcount - 1)
                fwrite(param->field,param->field_len,1,(fp == NULL?stdout:fp));
            }
            if(param->isFixlen) fwrite(param->record,param->return_len,1,(fp == NULL?stdout:fp));
          }
          trows = 0;
        }
      }
    }
    if (rows < param->asize) break;
  }
  if (trows % param->feedback != 0)
    print_feedback(trows);
  if(fp) fclose(fp);
 
  end_time=time(0);

  seconds = difftime(end_time,start_time);

  if(!param->isSTDOUT)
    fprintf((fp_log == NULL?stderr:fp_log),"         output file %s closed at %u rows. Elapsed time: %d min %d sec.\n", 
            tempbuf, trows, seconds/60, seconds%60);
  else
    if(fp_log) fprintf(fp_log,"         output file %s closed at %u rows. Elapsed time: %d min %d sec\n", 
            tempbuf, trows, seconds/60, seconds%60);
  if(fp_ctl) 
  {
    memset(tempbuf,0,128);
    sprintf(tempbuf,"%s_sqlldr.ctl",param->tabname);
    if(!param->isSTDOUT)
      fprintf((fp_log == NULL?stderr:fp_log),"         control file is %s\n", tempbuf);
    else
      if(fp_log) fprintf(fp_log,"         control file is %s\n", tempbuf);
  }
  if(!param->isSTDOUT) fprintf((fp_log == NULL?stderr:fp_log),"\n");
  fflush((fp_log == NULL?stderr:fp_log));

  if(iobuf) free(iobuf);
}

/* ----------------------------------------------------------------- */
/* get field and record char if it is hex code                       */
/* ----------------------------------------------------------------- */
int convert_option(const ub1 *src, ub1* dst, int mlen)
{
  int i,len,pos;
  ub1 c,c1,c2;

  i=pos=0;
  len = strlen(src);
 
/*  if(STRNCASECMP(src,"0x",2)) 
  {
    memcpy(dst,src,MIN(mlen,len));
    return MIN(mlen,len);
  }  
*/ 
  while(i<MIN(mlen,len))
  {
    if ( *(src+i) == '0')
    {
      if (i < len - 1)
      {
         c = *(src+i + 1);
         switch(c)
         {
           case 'x':
           case 'X':
             if (i < len - 3)
             {
               c1 = get_hexindex(*(src+i + 2));
               c2 = get_hexindex(*(src+i + 3));
               *(dst + pos) = (ub1)((c1 << 4) + c2);
               i=i+2;
             }
             else if (i < len - 2)
             {
               c1 = *(src+i + 2);
               *(dst + pos) = c1;
               i=i+1;
             }
               break;
            default:
               *(dst + pos) = c;
               break;
         }
         i = i + 2;
      }
      else
      {
        i ++;
      }
    }
    else
    {
      *(dst + pos) = *(src+i);
      i ++;
    }
    pos ++;
  }
  *(dst+pos) = '\0';

  return pos;
}

ub1  get_hexindex(char c)
{
  if ( c >='0' && c <='9') return c - '0';
  if ( c >='a' && c <='f') return 10 + c - 'a';
  if ( c >='A' && c <='F') return 10 + c - 'A';
  return 0;
}

/* ----------------------------------------------------------------- */
/* every 500000 rows print info to scr or fp_log                      */
/* ----------------------------------------------------------------- */
void print_feedback(ub4 row)
{
  time_t now = time(0);
  struct tm *ptm = localtime(&now);

  if(param->isSTDOUT && fp_log == NULL) return; 
  
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

/* ----------------------------------------------------------------- */
/* open files                                                        */
/* ----------------------------------------------------------------- */
FILE *open_file(const text *fname, text tempbuf[], int batch)
{
  FILE *fp=NULL;
  int i, j, len;
  time_t now = time(0);
  struct tm *ptm = localtime(&now);   
  
  len = strlen(fname);
  j = 0;
  for(i=0;i<len;i++)
  {
    if (*(fname+i) == '%')
    {
      i++;
      if (i < len)
      {
        switch(*(fname+i))
        {
          case 'Y':
          case 'y':
            j += sprintf(tempbuf+j, "%04d", ptm->tm_year + 1900);
            break;
          case 'M':
          case 'm':
            j += sprintf(tempbuf+j, "%02d", ptm->tm_mon + 1);
            break;
          case 'D':
          case 'd':
            j += sprintf(tempbuf+j, "%02d", ptm->tm_mday);
            break;
          case 'W':
          case 'w':
            j += sprintf(tempbuf+j, "%d", ptm->tm_wday);
            break;
          case 'B':
          case 'b':
            j += sprintf(tempbuf+j, "%d", batch);
            break;
          default:
            tempbuf[j++] = *(fname+i);
          break;
         }
       }
     }
     else
     {
       tempbuf[j++] = *(fname+i);
     }
   }
   tempbuf[j]=0;
   if (tempbuf[0] == '+')
     fp = fopen(tempbuf+1, "ab+");
   else
     fp = fopen(tempbuf, "wb+");
   return fp;
}

/* ----------------------------------------------------------------- */
/* write sqlldr control file                                         */
/* ----------------------------------------------------------------- */
void sqlldr_ctlfile(struct COLUMN *collist, ub2 numcols)
{
  text ctlfname[128]="";
  text tempbuf[1024];
  
  struct COLUMN *nextcol;
  int i,col;
  int totallen=1;

  nextcol = collist;
  if (strlen(param->tabname))
  {
    memset(ctlfname,0,128);
    sprintf(ctlfname,"%s_sqlldr.ctl",param->tabname);
    fp_ctl = open_file(ctlfname,tempbuf,0);

    if (fp_ctl)
    {
      fprintf(fp_ctl,"--\n");
      fprintf(fp_ctl,"-- Generated by tbuldr\n");
      fprintf(fp_ctl,"--\n");
      if (!param->header)
        fprintf(fp_ctl,"OPTIONS(BINDSIZE=%d,READSIZE=%d,ERRORS=-1,ROWS=50000)\n", param->buffer, param->buffer);
      else
        fprintf(fp_ctl,"OPTIONS(BINDSIZE=%d,READSIZE=%d,SKIP=1,ERRORS=-1,ROWS=50000)\n", param->buffer, param->buffer);
      fprintf(fp_ctl,"LOAD DATA\n");
      if(param->isFixlen)
      {
        fprintf(fp_ctl,"INFILE '%s' \"FIX %d\"\n", param->fname,row_totallen);
      }
      else
      {
        fprintf(fp_ctl,"INFILE '%s' \"STR X'", param->fname);
        for(i=0;i<strlen(param->record);i++) fprintf(fp_ctl,"%02x",param->record[i]);
        fprintf(fp_ctl,"'\"\n");
      }
      fprintf(fp_ctl,"%s INTO TABLE %s\n", param->tabmode, param->tabname);
      fprintf(fp_ctl,"FIELDS TERMINATED BY X'");
      for(i=0;i<strlen(param->field);i++) fprintf(fp_ctl,"%02x",param->field[i]);
      if(param->enclose_len)
      {
        fprintf(fp_ctl,"' ENCLOSED BY '%s",param->enclose);
      }
      fprintf(fp_ctl,"' TRAILING NULLCOLS \n");
      fprintf(fp_ctl,"(\n");
      
      for (col = 0; col < numcols; col++)
      {
        nextcol=nextcol->next;
      
        switch(nextcol->coltype)
        {
          case SQLT_DATE:
          case SQLT_DAT:
            if (param->isFixlen)
              fprintf(fp_ctl,"  %s POSITION(%d:%d) DATE \"YYYY-MM-DD HH24:MI:SS\"", 
                            nextcol->colname, totallen, totallen + nextcol->colwidth-1);
            else
              fprintf(fp_ctl,"  %s DATE \"YYYY-MM-DD HH24:MI:SS\"", nextcol->colname);
            break;
          case SQLT_TIMESTAMP: /* TIMESTAMP */
            if (param->isFixlen)
              fprintf(fp_ctl,"  %s POSITION(%d:%d) TIMESTAMP \"YYYY-MM-DD HH24:MI:SSXFF\"",
                     nextcol->colname , totallen, totallen + nextcol->colwidth-1);
            else
              fprintf(fp_ctl,"  %s TIMESTAMP \"YYYY-MM-DD HH24:MI:SSXFF\"", nextcol->colname);
            break;
          case SQLT_TIMESTAMP_TZ: /* TIMESTAMP WITH TIMEZONE */
            if (param->isFixlen)
              fprintf(fp_ctl,"  %s POSITION(%d:%d) TIMESTAMP WITH TIME ZONE \"YYYY-MM-DD HH24:MI:SSXFF TZH:TZM\"",
                     nextcol->colname , totallen, totallen + nextcol->colwidth-1);
            else
              fprintf(fp_ctl,"  %s TIMESTAMP WITH TIME ZONE \"YYYY-MM-DD HH24:MI:SSXFF TZH:TZM\"", nextcol->colname );
            break;
          case SQLT_LBI:  /* LONG RAW */
          case SQLT_BLOB:
            if (param->isFixlen)
              fprintf(fp_ctl," %s POSITION(%d:%d) CHAR(%d) ",
                     nextcol->colname , totallen, totallen + nextcol->colwidth-1,2 * nextcol->colwidth);
            else
              fprintf(fp_ctl,"  %s CHAR(%d) ", nextcol->colname, 2 * nextcol->colwidth);
            break;
          case SQLT_CLOB:  
            if (param->isFixlen)
              fprintf(fp_ctl," %s POSITION(%d:%d) CHAR(%d) ",
                     nextcol->colname , totallen, totallen + nextcol->colwidth-1);
            else
              fprintf(fp_ctl,"  %s CHAR(%d) ", nextcol->colname,  nextcol->colwidth);
            break;
          case SQLT_RDD:
            if (param->isFixlen)
              fprintf(fp_ctl, "  %s POSITION(%d:%d) CHAR(%d)", 
                nextcol->colname, totallen, totallen + nextcol->colwidth-1, nextcol->colwidth);
            else
              fprintf(fp_ctl, "  %s CHAR(%d)", nextcol->colname, nextcol->colwidth);
            break;
          case SQLT_INT:
          case SQLT_NUM:
            if (param->isFixlen)
              fprintf(fp_ctl,"  %s POSITION(%d:%d) CHAR(%d)",
                     nextcol->colname , totallen, totallen + nextcol->colwidth-1,nextcol->colwidth);
            else
              fprintf(fp_ctl,"  %s CHAR", nextcol->colname);
            break;
          case SQLT_FILE: /* BFILE */
            fprintf(fp_ctl,"  %s CHAR(%d)", nextcol->colname,param->lsize);
            break;
          default:
            if (param->isFixlen)
              fprintf(fp_ctl,"  %s POSITION(%d:%d) CHAR(%d)", nextcol->colname, 
                              totallen, totallen + nextcol->colwidth-1, nextcol->colwidth);
            else
              fprintf(fp_ctl,"  %s CHAR(%d)", nextcol->colname,nextcol->colwidth);
            break;
        }
        totallen = totallen + nextcol->colwidth;
        if(col<numcols-1)
          fprintf(fp_ctl,",\n");
      }
       fprintf(fp_ctl,"\n)\n");
    }
  }
}

/* ----------------------------------------------------------------- */
/* get parameters from cmdline                                       */
/* ----------------------------------------------------------------- */
int get_param(int argc,char **argv)
{
  text  tempbuf[1024];
  text  temptable[1024];
  text  *p_tmp;
  FILE  *fp_sql;
  sword i,j;

  param=(struct PARAM *) malloc(sizeof(struct PARAM));
  memset(param,0,sizeof(param));

  memcpy(param->fname,"uldrdata.txt",12);
  memcpy(param->tabmode,"INSERT",6);
  memcpy(param->field,",",1);
  memcpy(param->record,"\n",1);

  param->field_len = 1;
  param->return_len = 1;
  param->buffer = 16777216;  // 16MB
  param->feedback = 500000;
  param->asize = 50;
  param->lsize = 8192;

  for(i=0;i<argc;i++)
  { 
    if (STRNCASECMP("user=",argv[i],5)==0)
    {
      memcpy(param->connstr,(text *)argv[i]+5,MIN(strlen(argv[i]) - 5,127));
    }
    else if (STRNCASECMP("query=",argv[i],6)==0)
    {
      memcpy(param->query,argv[i]+6,MIN(strlen(argv[i]) - 6,1023));
      p_tmp=param->query;
      for(j=0;j<strlen(param->query);j++)
      {
        if(param->query[j]==' ') p_tmp++;
      }
     
      if(STRNCASECMP("select",p_tmp,6))
      { 
        memset(temptable,0,1024);
        memcpy(temptable,p_tmp,strlen(p_tmp));
        memset(param->query,0,1024);
        memcpy(param->query,"select * from ",14);
        strncat(param->query,temptable,strlen(temptable)); 
      }
    }
    else if (STRNCASECMP("file=",argv[i],5)==0)
    {
      memset(param->fname,0,128);
      memcpy(param->fname,argv[i]+5,MIN(strlen(argv[i]) - 5,127));
    }
    else if (STRNCASECMP("sql=",argv[i],4)==0)
    {
      memcpy(param->sqlfname,argv[i]+4,MIN(strlen(argv[i]) - 4,127));
    }
    else if (STRNCASECMP("field=",argv[i],6)==0)
    {
      memset(param->field,0,16);
      param->field_len=convert_option(argv[i]+6,param->field,MIN(strlen(argv[i]) - 6,15));
    }
    else if (STRNCASECMP("record=",argv[i],7)==0)
    {
      memset(param->record,0,16);
      param->return_len=convert_option(argv[i]+7,param->record,MIN(strlen(argv[i]) - 7,15));
    }
    else if (STRNCASECMP("enclose=",argv[i],8)==0)
    {
      memset(param->enclose,0,16);
      param->enclose_len=convert_option(argv[i]+8,param->enclose,MIN(strlen(argv[i]) - 8,15));
    }
    else if (STRNCASECMP("log=",argv[i],4)==0)
    {
      memcpy(param->logfile,argv[i]+4,MIN(strlen(argv[i]) - 4,127));
    }
    else if (STRNCASECMP("table=",argv[i],6)==0)
    {
      memcpy(param->tabname,argv[i]+6,MIN(strlen(argv[i]) - 6,127));
    }
    else if (STRNCASECMP("mode=",argv[i],5)==0)
    {
      memcpy(param->tabmode,argv[i]+5,MIN(strlen(argv[i]) - 5,15));
    }
    else if (STRNCASECMP("head=",argv[i],5)==0)
    {
      memset(tempbuf,0,16);
      memcpy(tempbuf,argv[i]+5,MIN(strlen(argv[i]) - 5,15));
      if (STRNCASECMP(tempbuf,"YES",3) == 0 
         || STRNCASECMP(tempbuf,"ON",3) == 0
         || STRNCASECMP(tempbuf,"1",3) == 0) param->header = 1;
    }
    else if (STRNCASECMP("sort=",argv[i],5)==0)
    {
      memset(tempbuf,0,1024);
      memcpy(tempbuf,argv[i]+5,MIN(strlen(argv[i]) - 5,254));
      param->ssize = atoi(tempbuf);
      if (param->ssize > 512) param->ssize = 512;
    }
    else if (STRNCASECMP("buffer=",argv[i],7)==0)
    {
      memset(tempbuf,0,1024);
      memcpy(tempbuf,argv[i]+7,MIN(strlen(argv[i]) - 7,254));
      param->buffer = atoi(tempbuf);
      if (param->buffer < 8) param->buffer = 8;
      if (param->ssize > 100) param->buffer = 100;
      param->buffer = param->buffer * 1048576;
    }
    else if (STRNCASECMP("long=",argv[i],5)==0)
    {
      memset(tempbuf,0,1024);
      memcpy(tempbuf,argv[i]+5,MIN(strlen(argv[i]) - 5,254));
      param->lsize = atoi(tempbuf);
      if (param->lsize < 100) param->lsize = 100;
      if (param->lsize > MAX_CLOB_SIZE) param->lsize = MAX_CLOB_SIZE;
    }
    else if (STRNCASECMP("array=",argv[i],6)==0)
    {
      memset(tempbuf,0,1024);
      memcpy(tempbuf,argv[i]+6,MIN(strlen(argv[i]) - 6,254));
      param->asize = atoi(tempbuf);
      if (param->asize < 5) param->asize = 5;
      if (param->asize > 3200) param->asize = 3200;
    }
    else if (STRNCASECMP("hash=",argv[i],5)==0)
    {
      memset(tempbuf,0,1024);
      memcpy(tempbuf,argv[i]+5,MIN(strlen(argv[i]) - 5,254));
      param->hsize = atoi(tempbuf);
      if (param->hsize > 512) param->hsize = 512;
    }
    else if (STRNCASECMP("read=",argv[i],5)==0)
    {
      memset(tempbuf,0,1024);
      memcpy(tempbuf,argv[i]+5,MIN(strlen(argv[i]) - 5,254));
      param->bsize = atoi(tempbuf);
      if (param->bsize > 512) param->bsize = 512;
    }
    else if (STRNCASECMP("batch=",argv[i],6)==0)
    {
      memset(tempbuf,0,1024);
      memcpy(tempbuf,argv[i]+6,MIN(strlen(argv[i]) - 6,254));
      param->batch = atoi(tempbuf);
      if (param->batch == 1) param->batch = 2;
    }
    else if (STRNCASECMP("serial=",argv[i],7)==0)
    {
      memset(tempbuf,0,1024);
      memcpy(tempbuf,argv[i]+7,MIN(strlen(argv[i]) - 7,254));
      param->serial = atoi(tempbuf);
    }
    else if (STRNCASECMP("trace=",argv[i],6)==0)
    {
      memset(tempbuf,0,1024);
      memcpy(tempbuf,argv[i]+6,MIN(strlen(argv[i]) - 6,254));
      param->trace = atoi(tempbuf);
    }
    else if (STRNCASECMP("feedback=",argv[i],9)==0)
    {
      memset(tempbuf,0,1024);
      memcpy(tempbuf,argv[i]+9,MIN(strlen(argv[i]) - 9,254));
      param->feedback = atoi(tempbuf);
    }
    else if (STRNCASECMP("form=",argv[i],5)==0)
    {
      memset(tempbuf,0,128);
      memcpy(tempbuf,argv[i]+5,MIN(strlen(argv[i]) - 5,128));
      if (STRNCASECMP(tempbuf,"YES",3) == 0 
         || STRNCASECMP(tempbuf,"ON",3) == 0
         || STRNCASECMP(tempbuf,"1",3) == 0) param->isForm = 1;
    }
    else if (STRNCASECMP("fixlen=",argv[i],7)==0)
    {
      memset(tempbuf,0,128);
      memcpy(tempbuf,argv[i]+7,MIN(strlen(argv[i]) - 7,16));
      if (STRNCASECMP(tempbuf,"YES",3) == 0
         || STRNCASECMP(tempbuf,"ON",3) == 0
         || STRNCASECMP(tempbuf,"1",3) == 0) param->isFixlen = 1;
    }
    else if (STRNCASECMP("debug=",argv[i],6)==0)
    {
      memset(tempbuf,0,1024);
      memcpy(tempbuf,argv[i]+6,MIN(strlen(argv[i]) - 6,254));
      if (STRNCASECMP(tempbuf,"YES",3) == 0 
         || STRNCASECMP(tempbuf,"ON",3) == 0
         || STRNCASECMP(tempbuf,"1",3) == 0) param->debug = 1;
    }
    else if (STRNCASECMP("-help",argv[i],4)==0)
    {
      print_help();
    }
  }

  if (strlen(param->sqlfname) > 0)
  {
    fp_sql = fopen(param->sqlfname,"r+");
    if (fp_sql != NULL)
    {
      memset(param->query,0,1024);
      while(!feof(fp_sql))
      {
        memset(tempbuf,0,1024);
        fgets(tempbuf,1023,fp_sql);
        strcat(param->query,tempbuf);
        strcat(param->query," ");
      }
      fclose(fp_sql);
    }
  }

  if(param->fname[0]=='-'){
    param->isSTDOUT=1;
  }
  
  if(!strlen(param->query)){
    print_help();
    return 1;
  }

  if (strlen(param->logfile))
  {
    fp_log = open_file(param->logfile,tempbuf,0);
  }

  if (param->asize * param->lsize > 104857600)
  {
    param->asize = 104857600/param->lsize;
    if (param->asize < 5) param->asize=5;
  }

  if(strlen(temptable) && !strlen(param->tabname)) 
  { 
    memcpy(param->tabname,temptable,strlen(temptable));
  }
 
  if (param->isForm) param->header=0;

  if (param->debug)
    printf("\narray:%d field_len:%d return_len:%d enclose_len:%d\n",param->asize,param->field_len,param->return_len,param->enclose_len);

  return 0;
}
```

