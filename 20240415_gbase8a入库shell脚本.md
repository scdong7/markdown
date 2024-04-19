# gbase8a入库shell脚本

```shell
#/bin/sh
##########################################################################################################
#20240415 sc created
#base on Gbase8a sftp load and export
#work host IP must single and test
##########################################################################################################
if [ $# -ne 2 ];then
    echo "参数错误"
    exit 1
fi
#task define
TASK_TYPE=$1 #0 load 1 export
TASK_DATE=$2 # data date
BATCH_NO=`date +%s`

#work home define
WORKDIR=`pwd`
WORKIP=$(hostname -I | awk '{ printf $1 }')
#scp user define
SFTP_USER=root
SFTP_PASS=Lzccb@1573
#Gbase8a db define
user=lrm
pass=Lrm@2024
host=192.16.40.150
port=5258
database=ALM_BASE
#other define
TEMP_CFG=$WORKDIR/tmp_cfg$TASK_TYPE
TEMP_LOG=$WORKDIR/tmp_log$TASK_TYPE
TEMP_CKLOG=$WORKDIR/tmp_cklog$TASK_TYPE

SEPARAOR="##"

#function define

GATBATCHCFG(){
    gccli -u$user -p$pass -h$host -P$port -D$database --skip-column-names --skip-line-numbers<<-EOF >$TEMP_CFG
        select concat(t.table_name,'##',t.check,'##',t.path) loads
        from alm_base.mas_load_cfg t
        left join (select * 
                from alm_base.mas_log_tasklog t 
                where t.batchno='$BATCH_NO' and t.batchdate='$TASK_DATE'
                    and t.status in ('0','1','2')
        )t2 on t.table_name=t2.table_name
        where t.status='0' and t.task_type='$TASK_TYPE'
            and t2.table_name is null;
EOF
}

LOAD(){
    cat /dev/null>$TEMP_LOG
    GATBATCHCFG
    for line in $(cat $TEMP_CFG |sort -R)
    do
        array=(${line//$SEPARAOR/})
        table_name=${array[0]}
        checkflag=${array[1]}
        path=${array[2]}

        task_id=`date +%s`
        if [ -f $path ]
        then
            filecnt=`cat $path |wc -l`
            gccli -u$user -p$pass -h$host -P$port -D$database --skip-column-names --skip-line-numbers<<-EOF >>$TEMP_LOG 2>&1
                insert into alm_base.mas_load_tasklog(batchno,batchdate,task_id,sdate,status,table_name,filecnt,checklog)
                values('$BATCH_NO','$TASK_DATE','$task_id',now(),0,'$table_name',$filecnt,0);
                commit;
                truncate table $table_name;
                load data infile '$sftp://${SFTP_USER}:${SFTP_PASS}@${WORKIP}${path}' into table $table_name character set gbk fields terminated by x'1c' autofill line terminated by '\n';
                update alm_base.mas_load_tasklog
                set edate=now(),status=1,
                    tabcnt=(select count(*) from $table_name)
                where batchno='$BATCH_NO' and batchdate='$TASK_DATE' and task_id='$task_id' and table_name='$table_name';
                commit;
EOF
        fi
        if [ $checkflag -ne 0 ]
        then 
            gccli -u$user -p$pass -h$host -P$port -D$database --skip-column-names --skip-line-numbers<<-EOF >>$TEMP_LOG 2>&1
                update alm_base.mas_load_tasklog
                set edate=now(),status=case when nvl(filecnt,0)<>nvl(tabcnt,0) then 3 else 2 end
                    checklog=case when nvl(filecnt,0)<>nvl(tabcnt,0) then 'file cnt is equal with tabcnt' else 2 end
                where batchno='$BATCH_NO' and batchdate='$TASK_DATE' and task_id='$task_id' and table_name='$table_name';
                commit;
EOF
        fi
    done
     gccli -u$user -p$pass -h$host -P$port -D$database --skip-column-names --skip-line-numbers<<-EOF >>$TEMP_CKLOG 2>&1
        select t.*
        from (
            select count(distinct t.table_name)-count(distinct t2.table_name) cnt
            from alm_base.mas_load_datacfg t
            left join (select * 
                       from alm_base.mas_load_tasklog t
                       where t.batchno='$BATCH_NO'  and t.batchdate='$TASK_DATE'
                            and t.status='2' and t.checklog='0'
            )t2 on t.table_name=t2.table_name
            where t.status='0' and t.task_type='$TASK_TYPE'
                and t.check<>'0'
        )t where t.cnt<>0;
EOF
    errors=`cat $TEMP_LOG |grep ERROR |wc -l`
    checks=`cat $TEMP_CKLOG |wc -l`
    echo "this load batch errors count is "$errors" and checks failed is "$checks
    if [ $errors -gt 0 ] || [ $checks -gt 0 ]
    then 
        sleep 300s
        LOAD
    else
        exit 0;
    fi
}

EXPORT(){
    cat /dev/null>$TEMP_LOG
    GATBATCHCFG
    for line in $(cat $TEMP_CFG |sort -R)
    do
        array=(${line//$SEPARAOR/})
        table_name=${array[0]}
        path=${array[2]}
  
        gccli -u$user -p$pass -h$host -P$port -D$database --skip-column-names --skip-line-numbers<<-EOF >>$TEMP_LOG 2>&1
	     set gbase_export_directory=0;
             select * from $table_name into outfile '$sftp://${SFTP_USER}:${SFTP_PASS}@${WORKIP}${path}' character set gbk fields terminated by x'1c' line terminated by '\n' null_value '' writemode by overwrites;
EOF
    done
    errors=`cat $TEMP_LOG |grep ERROR |wc -l`
    if [ $errors -gt 0 ]
    then 
        sleep 300s
        EXPORT
    else
        exit 0;
    fi
}

if [ $TASK_TYPE -eq 0 ]
then 
    LOAD
else 
    EXPORT
fi
```
