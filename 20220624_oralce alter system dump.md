```c
PURPOSE This article explain how to obtain a dump of the header information in the online redo log file(s), as well as obtaining selected information from the online or archived redo log files.

SCOPE & APPLICATION Informational 

You are working with Oracle Technical Support. As part of the diagnostic process, you have been asked to take a dump of the redo log files. The information in the logs is often used to help diagnose corruption issues.

The following commands will be used in this process: 
1. The 'alter session' command is used to dump redo headers. 
2. Use the 'alter system dump logfile' to dump log file contents. 

This command requires 'ALTER SYSTEM' system privilege. The database can be in mount, nomount or open state when the command is issued. An online log file or an archived log file can be dumped. It is even possible to dump a file from another database, as long as the operating systems are the same.

Output from the command is put into the session's trace file. 

The following ways of dumping a redo log file are covered:

1. To dump records based in DBA (Data Block Address) 
2. To dump records based on RBA (Redo Block Address) 
3. To dump records based on time 
4. To dump records based on layer and opcode 
5. Dump the file header information 
6. Dump an entire log file: 

1. To dump records based on DBA (Data Block Address) 
-------------------------------------------------- 
This will dump all redo records for the range of data blocks specified for a given file # and block # range.
From sqldba or svrmgr, issue the following command: 

ALTER SYSTEM DUMP LOGFILE 'filename' 
DBA MIN fileno . blockno 
DBA MAX fileno . blockno; 
Example:
======== 
ALTER SYSTEM DUMP LOGFILE 'u01/oracle/V7323/dbs/arch1_76.dbf' 
DBA MIN 5 . 31125 
DBA MAX 5 . 31150; 

This will cause all the changes to the specified range of data blocks to be dumped to the trace file. In the example given, all redo records for file #5, blocks 31125 thru 31150 are dumped.

2. To dump records based on RBA (Redo Block Address) 
------------------------------------------------- 
This will dump all redo records for the range of redo addresses specified for the given sequence number and block number.

Syntax:
ALTER SYSTEM DUMP LOGFILE 'filename' 
RBA MIN seqno . blockno 
RBA MAX seqno . blockno; 
Example:
ALTER SYSTEM DUMP LOGFILE 'u01/oracle/V7323/dbs/arch1_76.dbf' 
RBA MIN 2050 . 13255 
RBA MAX 2255 . 15555; 

3. To dump records based on time.
------------------------------ 
Using this option will cause redo records created within the time range specified to be dumped to the trace file.

From sqldba or svrmgr, issue the following command: 
ALTER SYSTEM DUMP LOGFILE 'filename' 
TIME MIN value
TIME MAX value; 

Example: 
======== 
ALTER SYSTEM DUMP LOGFILE 'u01/oracle/V7323/dbs/arch1_76.dbf' 
TIME MIN 299425687 
TIME MAX 299458800; 
Please Note: the time value is given in REDO DUMP TIME 

4. To dump records based on layer and opcode.
------------------------------------------ 
LAYER and OPCODE are used to dump all log records for a particular type of redo record, such as all dropped row pieces.
From sqldba or svrmgr, issue the following command: 

ALTER SYSTEM DUMP LOGFILE 'filename' 
LAYER value
OPCODE value; 
Example: 
======== 
ALTER SYSTEM DUMP LOGFILE 'u01/oracle/V7323/dbs/arch1_76.dbf' 
LAYER 11 
OPCODE 3; 

5. Dump the file header information:
---------------------------------
This will dump file header information for every online redo log file. From sqldba or svrmgr, issue the following command:
alter session set events 'immediate trace name redohdr level 10'; 

6. Dump an entire log file:
------------------------ 
From sqldba or svrmgr, issue the following command:
ALTER SYSTEM DUMP LOGFILE 'filename'; 
Please note: 
Fully qualify the filename, and include the single quotes.

Example:
======== 
ALTER SYSTEM DUMP LOGFILE 'u01/oracle/V7323/dbs/arch1_76.dbf';
```

