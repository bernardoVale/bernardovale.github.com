---
layout: post
title: RMAN Table Recovery
description: "Recuperação a nível de tabela com o Oracle 12c"
tags : [oracle, oracle12c, rman, recovery, pdb ]
image:
  background: block.jpg
comments: true
share: true
---

<figure >
	<img src="/images/table-recovery.jpg" alt="">
</figure>

# Objetivo

Uma das novas funcionalidades do Oracle 12c é a possibilidade de recuperar uma única tabela ou um set de tabelas em uma única ação de recovery via RMAN, detalhe, **com o banco no modo open e sem necessidade de colocar datafiles em modo offline**.

## Modos de Recuperação

O RMAN nos possibilita algumas variações do comando de recovery que pode ser útil em algum momento:

Você pode:

* Recuperar as tabelas com outro nome `REMAP TABLE`
* Recuperar as tabelas em uma tablespace diferente `REMAP TABLESPACE`
* Optar por apenas gerar um dump file e importar mais tarde `NOTABLEIMPORT`.

## Montagem do Cenário

Em nosso pequeno test case, iremos restaurar as tabelas do usuário scott em um Pluggable Database (PDB).

{% highlight sql %}

--Conecte no usuário SCOTT

--Marcar o horário para o recovery

select systimestamp from dual;

SYSTIMESTAMP
------------------------------------
10-APR-14 01.30.00.894835 PM -03:00


--Estado das tabelas antes do truncate
select count(*) from emp;

  COUNT(*)
----------
	14

select count(*) from dept;

  COUNT(*)
----------
	 4

select count(*) from bonus;

  COUNT(*)
----------
	 0

select count(*) from salgrade;

  COUNT(*)
----------
	 5

--USE ESSA PROCEDURE PARA DESABILITAR AS CONSTRAINTS
BEGIN
  FOR c IN
  (SELECT c.owner, c.table_name, c.constraint_name
   FROM user_constraints c, user_tables t
   WHERE c.table_name = t.table_name
   AND c.status = 'ENABLED'
   ORDER BY c.constraint_type DESC)
  LOOP
    dbms_utility.exec_ddl_statement('alter table "' || c.owner 
	|| '"."' || c.table_name || '" disable constraint ' || c.constraint_name);
  END LOOP;
END;
/

PL/SQL procedure successfully completed.

--PROCEDURE PARA TRUNCAR TODAS AS TABELAS

BEGIN
  FOR c IN
  (SELECT table_name from user_tables)
  LOOP
    dbms_utility.exec_ddl_statement('truncate table ' || c.table_name );
  END LOOP;
END;
/

PL/SQL procedure successfully completed.

--TODAS AS TABELAS FORAM TRUNCADAS

SQL> select count(*) from emp;

  COUNT(*)
----------
	 0

SQL> select count(*) from dept;

  COUNT(*)
----------
	 0

SQL> select count(*) from bonus;

  COUNT(*)
----------
	 0

SQL> select count(*) from salgrade;

  COUNT(*)
----------
	 0

SQL> select systimestamp from dual;

SYSTIMESTAMP
-------------------------------------
10-APR-14 02.02.00.894835 PM -03:00

{% endhighlight %}

> Caso queira recuperar as tabelas com os mesmos nomes, você deverá exclui-las primeiro.
No exemplo, iremos importar as tabelas com outro nome usando a cláusula `REMAP TABLE`.
{% highlight sql %}
--CONECTADO COMO TARGET DATABASE NO RMAN

RMAN> recover table 'SCOTT'.'EMP', 'SCOTT'.'DEPT', 'SCOTT'.'SALGRADE', 'SCOTT'.'BONUS'
of pluggable database pdbora
until time "TO_DATE ('10/04/2014 01:30:00','DD/MM/yyyy HH24:MI:SS')"
auxiliary destination '/u01/app/oracle/aux'
remap table 'SCOTT'.'EMP':'REST_EMP', 'SCOTT'.'DEPT':'REST_DEPT', 'SCOTT'.'SALGRADE':'REST_SALGRADE', 'SCOTT'.'BONUS':'REST_BONUS';
2> 3> 4> 5> 
Starting recover at 10-APR-14
using channel ORA_DISK_1
using channel ORA_DISK_2
using channel ORA_DISK_3
using channel ORA_DISK_4
RMAN-05026: WARNING: presuming following set of tablespaces applies to specified Point-in-Time

List of tablespaces expected to have UNDO segments
Tablespace SYSTEM
Tablespace UNDOTBS1

Creating automatic instance, with SID='tdfp'

initialization parameters used for automatic instance:
db_name=CDBORA
db_unique_name=tdfp_pitr_pdbora_CDBORA
compatible=12.1.0.0.0
db_block_size=8192
db_files=200
sga_target=1G
processes=80
diagnostic_dest=/u01/app/oracle
db_create_file_dest=/u01/app/oracle/aux
log_archive_dest_1='location=/u01/app/oracle/aux'
enable_pluggable_database=true
_clone_one_pdb_recovery=true
#No auxiliary parameter file used


starting up automatic instance CDBORA

Oracle instance started

Total System Global Area    1068937216 bytes

Fixed Size                     2296576 bytes
Variable Size                281019648 bytes
Database Buffers             780140544 bytes
Redo Buffers                   5480448 bytes
Automatic instance created

contents of Memory Script:
{
# set requested point in time
set until  time "TO_DATE ('10/04/2014 01:30:00','DD/MM/yyyy HH24:MI:SS')";
# restore the controlfile
restore clone controlfile;
# mount the controlfile
sql clone 'alter database mount clone database';
# archive current online log 
sql 'alter system archive log current';
}
executing Memory Script

executing command: SET until clause

Starting restore at 10-APR-14
allocated channel: ORA_AUX_DISK_1
channel ORA_AUX_DISK_1: SID=75 device type=DISK
allocated channel: ORA_AUX_DISK_2
channel ORA_AUX_DISK_2: SID=82 device type=DISK
allocated channel: ORA_AUX_DISK_3
channel ORA_AUX_DISK_3: SID=12 device type=DISK
allocated channel: ORA_AUX_DISK_4
channel ORA_AUX_DISK_4: SID=83 device type=DISK

channel ORA_AUX_DISK_1: starting datafile backup set restore
channel ORA_AUX_DISK_1: restoring control file
channel ORA_AUX_DISK_1: reading from backup piece /u01/app/oracle/backup/rman/ctl_c-774216564-20140408-02.bkp
channel ORA_AUX_DISK_1: piece handle=/u01/app/oracle/backup/rman/ctl_c-774216564-20140408-02.bkp tag=TAG20140408T194951
channel ORA_AUX_DISK_1: restored backup piece 1
channel ORA_AUX_DISK_1: restore complete, elapsed time: 00:00:01
output file name=/u01/app/oracle/aux/CDBORA/controlfile/o1_mf_9nfotjfm_.ctl
Finished restore at 10-APR-14

sql statement: alter database mount clone database

sql statement: alter system archive log current

contents of Memory Script:
{
# set requested point in time
set until  time "TO_DATE ('10/04/2014 01:30:00','DD/MM/yyyy HH24:MI:SS')";
# set destinations for recovery set and auxiliary set datafiles
set newname for clone datafile  1 to new;
set newname for clone datafile  4 to new;
set newname for clone datafile  3 to new;
set newname for clone datafile  26 to new;
set newname for clone datafile  27 to new;
set newname for clone tempfile  1 to new;
set newname for clone tempfile  3 to new;
# switch all tempfiles
switch clone tempfile all;
# restore the tablespaces in the recovery set and the auxiliary set
restore clone datafile  1, 4, 3, 26, 27;
switch clone datafile all;
}
executing Memory Script

executing command: SET until clause

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

executing command: SET NEWNAME

renamed tempfile 1 to /u01/app/oracle/aux/CDBORA/datafile/o1_mf_temp_%u_.tmp in control file
renamed tempfile 3 to /u01/app/oracle/aux/CDBORA/datafile/o1_mf_temp_%u_.tmp in control file

Starting restore at 10-APR-14
using channel ORA_AUX_DISK_1
using channel ORA_AUX_DISK_2
using channel ORA_AUX_DISK_3
using channel ORA_AUX_DISK_4

channel ORA_AUX_DISK_1: restoring datafile 00001
input datafile copy RECID=7 STAMP=844372175 file name=/u01/app/oracle/fast_recovery_area/CDBORA/datafile/o1_mf_system_9n8z7zry_.dbf
destination for restore of datafile 00001: /u01/app/oracle/aux/CDBORA/datafile/o1_mf_system_%u_.dbf
channel ORA_AUX_DISK_2: starting datafile backup set restore
channel ORA_AUX_DISK_2: specifying datafile(s) to restore from backup set
channel ORA_AUX_DISK_2: restoring datafile 00027 to /u01/app/oracle/aux/CDBORA/datafile/o1_mf_sysaux_%u_.dbf
channel ORA_AUX_DISK_2: reading from backup piece /u01/app/oracle/fast_recovery_area/CDBORA/F663CD84A6346D43E043CA00C80AD86E/backupset/2014_04_08/o1_mf_nnndf_TAG20140408T190318_9n8wk8sf_.bkp
channel ORA_AUX_DISK_3: starting datafile backup set restore
channel ORA_AUX_DISK_3: specifying datafile(s) to restore from backup set
channel ORA_AUX_DISK_3: restoring datafile 00003 to /u01/app/oracle/aux/CDBORA/datafile/o1_mf_sysaux_%u_.dbf
channel ORA_AUX_DISK_3: reading from backup piece /u01/app/oracle/fast_recovery_area/CDBORA/backupset/2014_04_08/o1_mf_nnndf_TAG20140408T190318_9n8wk734_.bkp
channel ORA_AUX_DISK_4: starting datafile backup set restore
channel ORA_AUX_DISK_4: specifying datafile(s) to restore from backup set
channel ORA_AUX_DISK_4: restoring datafile 00004 to /u01/app/oracle/aux/CDBORA/datafile/o1_mf_undotbs1_%u_.dbf
channel ORA_AUX_DISK_4: reading from backup piece /u01/app/oracle/fast_recovery_area/CDBORA/backupset/2014_04_08/o1_mf_nnndf_TAG20140408T190318_9n8wn39o_.bkp
channel ORA_AUX_DISK_4: piece handle=/u01/app/oracle/fast_recovery_area/CDBORA/backupset/2014_04_08/o1_mf_nnndf_TAG20140408T190318_9n8wn39o_.bkp tag=TAG20140408T190318
channel ORA_AUX_DISK_4: restored backup piece 1
channel ORA_AUX_DISK_4: restore complete, elapsed time: 00:00:03
channel ORA_AUX_DISK_4: starting datafile backup set restore
channel ORA_AUX_DISK_4: specifying datafile(s) to restore from backup set
channel ORA_AUX_DISK_4: restoring datafile 00026 to /u01/app/oracle/aux/CDBORA/datafile/o1_mf_system_%u_.dbf
channel ORA_AUX_DISK_4: reading from backup piece /u01/app/oracle/fast_recovery_area/CDBORA/F663CD84A6346D43E043CA00C80AD86E/backupset/2014_04_08/o1_mf_nnndf_TAG20140408T190318_9n8wmndg_.bkp
channel ORA_AUX_DISK_1: copied datafile copy of datafile 00001
output file name=/u01/app/oracle/aux/CDBORA/datafile/o1_mf_system_9nfotpbc_.dbf RECID=9 STAMP=844526391
channel ORA_AUX_DISK_4: piece handle=/u01/app/oracle/fast_recovery_area/CDBORA/F663CD84A6346D43E043CA00C80AD86E/backupset/2014_04_08/o1_mf_nnndf_TAG20140408T190318_9n8wmndg_.bkp tag=TAG20140408T190318
channel ORA_AUX_DISK_4: restored backup piece 1
channel ORA_AUX_DISK_4: restore complete, elapsed time: 00:00:25
channel ORA_AUX_DISK_2: piece handle=/u01/app/oracle/fast_recovery_area/CDBORA/F663CD84A6346D43E043CA00C80AD86E/backupset/2014_04_08/o1_mf_nnndf_TAG20140408T190318_9n8wk8sf_.bkp tag=TAG20140408T190318
channel ORA_AUX_DISK_2: restored backup piece 1
channel ORA_AUX_DISK_2: restore complete, elapsed time: 00:00:48
channel ORA_AUX_DISK_3: piece handle=/u01/app/oracle/fast_recovery_area/CDBORA/backupset/2014_04_08/o1_mf_nnndf_TAG20140408T190318_9n8wk734_.bkp tag=TAG20140408T190318
channel ORA_AUX_DISK_3: restored backup piece 1
channel ORA_AUX_DISK_3: restore complete, elapsed time: 00:00:58
Finished restore at 10-APR-14

datafile 1 switched to datafile copy
input datafile copy RECID=13 STAMP=844526432 file name=/u01/app/oracle/aux/CDBORA/datafile/o1_mf_system_9nfotpbc_.dbf
datafile 4 switched to datafile copy
input datafile copy RECID=14 STAMP=844526432 file name=/u01/app/oracle/aux/CDBORA/datafile/o1_mf_undotbs1_9nfotpg8_.dbf
datafile 3 switched to datafile copy
input datafile copy RECID=15 STAMP=844526432 file name=/u01/app/oracle/aux/CDBORA/datafile/o1_mf_sysaux_9nfotpg3_.dbf
datafile 26 switched to datafile copy
input datafile copy RECID=16 STAMP=844526432 file name=/u01/app/oracle/aux/CDBORA/datafile/o1_mf_system_9nfotsmp_.dbf
datafile 27 switched to datafile copy
input datafile copy RECID=17 STAMP=844526432 file name=/u01/app/oracle/aux/CDBORA/datafile/o1_mf_sysaux_9nfotphp_.dbf

contents of Memory Script:
{
# set requested point in time
set until  time "TO_DATE ('10/04/2014 01:30:00','DD/MM/yyyy HH24:MI:SS')";
# online the datafiles restored or switched
sql clone "alter database datafile  1 online";
sql clone "alter database datafile  4 online";
sql clone "alter database datafile  3 online";
sql clone 'PDBORA' "alter database datafile 
 26 online";
sql clone 'PDBORA' "alter database datafile 
 27 online";
# recover and open database read only
recover clone database tablespace  "SYSTEM", "UNDOTBS1", "SYSAUX", "PDBORA":"SYSTEM", "PDBORA":"SYSAUX";
sql clone 'alter database open read only';
}
executing Memory Script

executing command: SET until clause

sql statement: alter database datafile  1 online

sql statement: alter database datafile  4 online

sql statement: alter database datafile  3 online

sql statement: alter database datafile  26 online

sql statement: alter database datafile  27 online

Starting recover at 10-APR-14
using channel ORA_AUX_DISK_1
using channel ORA_AUX_DISK_2
using channel ORA_AUX_DISK_3
using channel ORA_AUX_DISK_4

starting media recovery

archived log for thread 1 with sequence 20 is already on disk as file /u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_08/o1_mf_1_20_9n8wncom_.arc
archived log for thread 1 with sequence 21 is already on disk as file /u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_08/o1_mf_1_21_9n8wxbom_.arc
archived log for thread 1 with sequence 22 is already on disk as file /u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_08/o1_mf_1_22_9n8wxgsh_.arc
archived log for thread 1 with sequence 23 is already on disk as file /u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_08/o1_mf_1_23_9n96y43r_.arc
archived log for thread 1 with sequence 24 is already on disk as file /u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_09/o1_mf_1_24_9n9xm2by_.arc
archived log for thread 1 with sequence 25 is already on disk as file /u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_09/o1_mf_1_25_9ncf7z9b_.arc
archived log for thread 1 with sequence 26 is already on disk as file /u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_09/o1_mf_1_26_9ncv9yv6_.arc
archived log for thread 1 with sequence 27 is already on disk as file /u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_10/o1_mf_1_27_9nf1yrnt_.arc
archived log file name=/u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_08/o1_mf_1_20_9n8wncom_.arc thread=1 sequence=20
archived log file name=/u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_08/o1_mf_1_21_9n8wxbom_.arc thread=1 sequence=21
archived log file name=/u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_08/o1_mf_1_22_9n8wxgsh_.arc thread=1 sequence=22
archived log file name=/u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_08/o1_mf_1_23_9n96y43r_.arc thread=1 sequence=23
archived log file name=/u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_09/o1_mf_1_24_9n9xm2by_.arc thread=1 sequence=24
archived log file name=/u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_09/o1_mf_1_25_9ncf7z9b_.arc thread=1 sequence=25
archived log file name=/u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_09/o1_mf_1_26_9ncv9yv6_.arc thread=1 sequence=26
archived log file name=/u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_10/o1_mf_1_27_9nf1yrnt_.arc thread=1 sequence=27
media recovery complete, elapsed time: 00:00:19
Finished recover at 10-APR-14

sql statement: alter database open read only

contents of Memory Script:
{
sql clone 'alter pluggable database  PDBORA open read only';
}
executing Memory Script

sql statement: alter pluggable database  PDBORA open read only

contents of Memory Script:
{
   sql clone "create spfile from memory";
   shutdown clone immediate;
   startup clone nomount;
   sql clone "alter system set  control_files = 
  ''/u01/app/oracle/aux/CDBORA/controlfile/o1_mf_9nfotjfm_.ctl'' comment=
 ''RMAN set'' scope=spfile";
   shutdown clone immediate;
   startup clone nomount;
# mount database
sql clone 'alter database mount clone database';
}
executing Memory Script

sql statement: create spfile from memory

database closed
database dismounted
Oracle instance shut down

connected to auxiliary database (not started)
Oracle instance started

Total System Global Area    1068937216 bytes

Fixed Size                     2296576 bytes
Variable Size                285213952 bytes
Database Buffers             775946240 bytes
Redo Buffers                   5480448 bytes

sql statement: alter system set  control_files =   ''/u01/app/oracle/aux/CDBORA/controlfile/o1_mf_9nfotjfm_.ctl'' comment= ''RMAN set'' scope=spfile

Oracle instance shut down

connected to auxiliary database (not started)
Oracle instance started

Total System Global Area    1068937216 bytes

Fixed Size                     2296576 bytes
Variable Size                285213952 bytes
Database Buffers             775946240 bytes
Redo Buffers                   5480448 bytes

sql statement: alter database mount clone database

contents of Memory Script:
{
# set requested point in time
set until  time "TO_DATE ('10/04/2014 01:30:00','DD/MM/yyyy HH24:MI:SS')";
# set destinations for recovery set and auxiliary set datafiles
set newname for datafile  28 to new;
# restore the tablespaces in the recovery set and the auxiliary set
restore clone datafile  28;
switch clone datafile all;
}
executing Memory Script

executing command: SET until clause

executing command: SET NEWNAME

Starting restore at 10-APR-14
allocated channel: ORA_AUX_DISK_1
channel ORA_AUX_DISK_1: SID=11 device type=DISK
allocated channel: ORA_AUX_DISK_2
channel ORA_AUX_DISK_2: SID=82 device type=DISK
allocated channel: ORA_AUX_DISK_3
channel ORA_AUX_DISK_3: SID=12 device type=DISK
allocated channel: ORA_AUX_DISK_4
channel ORA_AUX_DISK_4: SID=84 device type=DISK

channel ORA_AUX_DISK_1: starting datafile backup set restore
channel ORA_AUX_DISK_1: specifying datafile(s) to restore from backup set
channel ORA_AUX_DISK_1: restoring datafile 00028 to /u01/app/oracle/aux/TDFP_PITR_PDBORA_CDBORA/datafile/o1_mf_users_%u_.dbf
channel ORA_AUX_DISK_1: reading from backup piece /u01/app/oracle/fast_recovery_area/CDBORA/F663CD84A6346D43E043CA00C80AD86E/backupset/2014_04_08/o1_mf_nnndf_TAG20140408T190318_9n8wn4k6_.bkp
channel ORA_AUX_DISK_1: piece handle=/u01/app/oracle/fast_recovery_area/CDBORA/F663CD84A6346D43E043CA00C80AD86E/backupset/2014_04_08/o1_mf_nnndf_TAG20140408T190318_9n8wn4k6_.bkp tag=TAG20140408T190318
channel ORA_AUX_DISK_1: restored backup piece 1
channel ORA_AUX_DISK_1: restore complete, elapsed time: 00:00:01
Finished restore at 10-APR-14

datafile 28 switched to datafile copy
input datafile copy RECID=19 STAMP=844526487 file name=/u01/app/oracle/aux/TDFP_PITR_PDBORA_CDBORA/datafile/o1_mf_users_9nfoy6x0_.dbf

contents of Memory Script:
{
# set requested point in time
set until  time "TO_DATE ('10/04/2014 01:30:00','DD/MM/yyyy HH24:MI:SS')";
# online the datafiles restored or switched
sql clone 'PDBORA' "alter database datafile 
 28 online";
# recover and open resetlogs
recover clone database tablespace  "PDBORA":"USERS", "SYSTEM", "UNDOTBS1", "SYSAUX", "PDBORA":"SYSTEM", "PDBORA":"SYSAUX" delete archivelog;
alter clone database open resetlogs;
}
executing Memory Script

executing command: SET until clause

sql statement: alter database datafile  28 online

Starting recover at 10-APR-14
using channel ORA_AUX_DISK_1
using channel ORA_AUX_DISK_2
using channel ORA_AUX_DISK_3
using channel ORA_AUX_DISK_4

starting media recovery

archived log for thread 1 with sequence 23 is already on disk as file /u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_08/o1_mf_1_23_9n96y43r_.arc
archived log for thread 1 with sequence 24 is already on disk as file /u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_09/o1_mf_1_24_9n9xm2by_.arc
archived log for thread 1 with sequence 25 is already on disk as file /u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_09/o1_mf_1_25_9ncf7z9b_.arc
archived log for thread 1 with sequence 26 is already on disk as file /u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_09/o1_mf_1_26_9ncv9yv6_.arc
archived log for thread 1 with sequence 27 is already on disk as file /u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_10/o1_mf_1_27_9nf1yrnt_.arc
archived log file name=/u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_08/o1_mf_1_23_9n96y43r_.arc thread=1 sequence=23
archived log file name=/u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_09/o1_mf_1_24_9n9xm2by_.arc thread=1 sequence=24
archived log file name=/u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_09/o1_mf_1_25_9ncf7z9b_.arc thread=1 sequence=25
archived log file name=/u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_09/o1_mf_1_26_9ncv9yv6_.arc thread=1 sequence=26
archived log file name=/u01/app/oracle/fast_recovery_area/CDBORA/archivelog/2014_04_10/o1_mf_1_27_9nf1yrnt_.arc thread=1 sequence=27
media recovery complete, elapsed time: 00:00:01
Finished recover at 10-APR-14

database opened

contents of Memory Script:
{
sql clone 'alter pluggable database  PDBORA open';
}
executing Memory Script

sql statement: alter pluggable database  PDBORA open

contents of Memory Script:
{
# create directory for datapump import
sql 'PDBORA' "create or replace directory 
TSPITR_DIROBJ_DPDIR as ''
/u01/app/oracle/aux''";
# create directory for datapump export
sql clone 'PDBORA' "create or replace directory 
TSPITR_DIROBJ_DPDIR as ''
/u01/app/oracle/aux''";
}
executing Memory Script

sql statement: create or replace directory TSPITR_DIROBJ_DPDIR as ''/u01/app/oracle/aux''

sql statement: create or replace directory TSPITR_DIROBJ_DPDIR as ''/u01/app/oracle/aux''

Performing export of tables...
   EXPDP> Starting "SYS"."TSPITR_EXP_tdfp_cmwz":  
   EXPDP> Estimate in progress using BLOCKS method...
   EXPDP> Processing object type TABLE_EXPORT/TABLE/TABLE_DATA
   EXPDP> Total estimation using BLOCKS method: 192 KB
   EXPDP> Processing object type TABLE_EXPORT/TABLE/TABLE
   EXPDP> Processing object type TABLE_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
   EXPDP> Processing object type TABLE_EXPORT/TABLE/STATISTICS/MARKER
   EXPDP> ORA-39127: unexpected error from call to export_string :=SYS.DBMS_TRANSFORM_EXIMP.INSTANCE_INFO_EXP('AQ$_ORDERS_QUEUETABLE_S','IX',1,1,'12.01.00.00.00',newblock) 
ORA-00376: file 29 cannot be read at this time
ORA-01110: data file 29: '/u01/app/oracle/oradata/cdbora/pdbora/example01.dbf'
ORA-06512: at "SYS.DBMS_TRANSFORM_EXIMP", line 197
ORA-06512: at line 1
ORA-06512: at "SYS.DBMS_METADATA", line 9901
ORA-39127: unexpected error from call to export_string :=SYS.DBMS_TRANSFORM_EXIMP.INSTANCE_INFO_EXP('AQ$_STREAMS_QUEUE_TABLE_S','IX',1,1,'12.01.00.00.00',newblock) 
ORA-00376: file 29 cannot be read at this time
ORA-01110: data file 29: '/u01/app/oracle/oradata/cdbora/pdbora/example01.dbf'
ORA-06512: at "SYS.DBMS_TRANSFORM_EXIMP", line 197
ORA-06512: at line 1
ORA-06512: at "SYS.DBMS_METADATA", line 9901
   EXPDP> . . exported "SCOTT"."DEPT"                              6.007 KB       4 rows
   EXPDP> . . exported "SCOTT"."EMP"                               8.757 KB      14 rows
   EXPDP> . . exported "SCOTT"."SALGRADE"                          5.937 KB       5 rows
   EXPDP> . . exported "SCOTT"."BONUS"                                 0 KB       0 rows
   EXPDP> Master table "SYS"."TSPITR_EXP_tdfp_cmwz" successfully loaded/unloaded
   EXPDP> ******************************************************************************
   EXPDP> Dump file set for SYS.TSPITR_EXP_tdfp_cmwz is:
   EXPDP>   /u01/app/oracle/aux/tspitr_tdfp_43207.dmp
   EXPDP> Job "SYS"."TSPITR_EXP_tdfp_cmwz" completed with 2 error(s) at Thu Apr 10 14:43:15 2014 elapsed 0 00:01:20
Export completed


contents of Memory Script:
{
# shutdown clone before import
shutdown clone abort
}
executing Memory Script

Oracle instance shut down

Performing import of tables...
   IMPDP> Master table "SYS"."TSPITR_IMP_tdfp_wtuq" successfully loaded/unloaded
   IMPDP> Starting "SYS"."TSPITR_IMP_tdfp_wtuq":  
   IMPDP> Processing object type TABLE_EXPORT/TABLE/TABLE
   IMPDP> Processing object type TABLE_EXPORT/TABLE/TABLE_DATA
   IMPDP> . . imported "SCOTT"."REST_DEPT"                         6.007 KB       4 rows
   IMPDP> . . imported "SCOTT"."REST_EMP"                          8.757 KB      14 rows
   IMPDP> . . imported "SCOTT"."REST_SALGRADE"                     5.937 KB       5 rows
   IMPDP> . . imported "SCOTT"."REST_BONUS"                            0 KB       0 rows
   IMPDP> Processing object type TABLE_EXPORT/TABLE/STATISTICS/TABLE_STATISTICS
   IMPDP> Processing object type TABLE_EXPORT/TABLE/STATISTICS/MARKER
   IMPDP> Job "SYS"."TSPITR_IMP_tdfp_wtuq" successfully completed at Thu Apr 10 14:43:45 2014 elapsed 0 00:00:22
Import completed


Removing automatic instance
Automatic instance removed
auxiliary instance file /u01/app/oracle/aux/CDBORA/datafile/o1_mf_temp_9nfoxcvg_.tmp deleted
auxiliary instance file /u01/app/oracle/aux/CDBORA/datafile/o1_mf_temp_9nfox5s1_.tmp deleted
auxiliary instance file /u01/app/oracle/aux/TDFP_PITR_PDBORA_CDBORA/onlinelog/o1_mf_3_9nfoycwj_.log deleted
auxiliary instance file /u01/app/oracle/aux/TDFP_PITR_PDBORA_CDBORA/onlinelog/o1_mf_2_9nfoycj8_.log deleted
auxiliary instance file /u01/app/oracle/aux/TDFP_PITR_PDBORA_CDBORA/onlinelog/o1_mf_1_9nfoy9kr_.log deleted
auxiliary instance file /u01/app/oracle/aux/TDFP_PITR_PDBORA_CDBORA/datafile/o1_mf_users_9nfoy6x0_.dbf deleted
auxiliary instance file /u01/app/oracle/aux/CDBORA/datafile/o1_mf_sysaux_9nfotphp_.dbf deleted
auxiliary instance file /u01/app/oracle/aux/CDBORA/datafile/o1_mf_system_9nfotsmp_.dbf deleted
auxiliary instance file /u01/app/oracle/aux/CDBORA/datafile/o1_mf_sysaux_9nfotpg3_.dbf deleted
auxiliary instance file /u01/app/oracle/aux/CDBORA/datafile/o1_mf_undotbs1_9nfotpg8_.dbf deleted
auxiliary instance file /u01/app/oracle/aux/CDBORA/datafile/o1_mf_system_9nfotpbc_.dbf deleted
auxiliary instance file /u01/app/oracle/aux/CDBORA/controlfile/o1_mf_9nfotjfm_.ctl deleted
auxiliary instance file tspitr_tdfp_43207.dmp deleted
Finished recover at 10-APR-14

{% endhighlight %}

Optei por deixar todas as linhas do RMAN para fazermos alguns comentários sobre esse método de recovery.

Esse modo de recovery assemelha-se muito ao recovery tablespace

* Oracle cria uma instância auxiliar com as tablespaces SYSTEM,UNDO,TEMP e o set de tablespaces necessárias para o recovery.

* Faz um point-in-time recovery na nova instância usando como parâmetro a nossa especificação.

* Gera um datapump de todas as tabelas especificadas no comando

* Importa os dados no banco target com exceção da clausula NOTABLEIMPORT

# Conclusão

Mais uma novidade do Oracle 12c que traz imenso alívio para os DBA's acostumados a resolver esse tipo de problema. Nossa vantagem agora é que podemos fazer um recovery sem a necessidade de voltar toda a tablespace.