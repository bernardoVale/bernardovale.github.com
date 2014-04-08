---
layout: post
title: PluggableDB from PDB$SEED
description: "Criando um novo Pluggable Database a partir do PDB$SEED"
tags : [oracle12c, 12c, pdb$seed, pdb, cdb, ORA-65016 ]
comments: true
share: true
---

## Objetivo

Demonstrar uma das formar de criar os novos `Pluggable Database` através do PDB default `PDB$SEED` como base para criar um novo PDB do zero.

#PRE-CHECK

* Deve estar conectado a um Container Database (CDB)
* O CDB deve estar aberto e em Read write mode
* O usuário deve estar conectado ao CDB$ROOT
* O usuário deve ter o system privillege `create pluggable database`
* O PDB deve ter um nome único.

#Criando o PDB

Após checar todos os itens acima, podemos criar o novo PDB:

{% highlight sql %}

CREATE PLUGGABLE DATABASE testepdb
ADMIN USER testeadm IDENTIFIED BY testeadm;

ADMIN USER testeadm IDENTIFIED BY testeadm
                                         *
ERROR at line 2:
ORA-65016: FILE_NAME_CONVERT must be specified

--Deve-se adicionar a clausula de conversão mesmo
-- com o seed pdb

CREATE PLUGGABLE DATABASE testepdb
ADMIN USER testeadm IDENTIFIED BY testeadm
FILE_NAME_CONVERT = ('/pdbseed/', '/testepdb/');

Pluggable database created.
{% endhighlight %}

{% highlight bash %}
#A clausula do file_name_convert teve como base o
# diretório pdbseed onde constam os datafiles desse pdb
# que foram usados para a criação do testepdb
[oracle@bernardo cdbora]$ ls -l
total 1841640
-rw-r-----. 1 oracle oinstall  17973248 Apr  7 19:06 control01.ctl
drwxr-x---. 2 oracle oinstall      4096 Apr  6 14:08 pdbora
drwxr-x---. 2 oracle oinstall      4096 Apr  6 14:04 pdbseed
-rw-r-----. 1 oracle oinstall  52429312 Apr  7 03:00 redo01.log
-rw-r-----. 1 oracle oinstall  52429312 Apr  7 16:00 redo02.log
-rw-r-----. 1 oracle oinstall  52429312 Apr  7 19:06 redo03.log
-rw-r-----. 1 oracle oinstall 754982912 Apr  7 19:06 sysaux01.dbf
-rw-r-----. 1 oracle oinstall 807411712 Apr  7 19:06 system01.dbf
-rw-r-----. 1 oracle oinstall  62922752 Apr  7 19:01 temp01.dbf
drwxr-x---. 2 oracle oinstall      4096 Apr  7 18:50 testepdb
-rw-r-----. 1 oracle oinstall 141565952 Apr  7 19:05 undotbs01.dbf
-rw-r-----. 1 oracle oinstall   5251072 Apr  7 16:05 users01.dbf
[oracle@bernardo cdbora]$ ls -l pdbseed/
total 860240
-rw-r-----. 1 oracle oinstall  20979712 Apr  6 14:07 pdbseed_temp01.dbf
-rw-r-----. 1 oracle oinstall 618668032 Apr  6 14:07 sysaux01.dbf
-rw-r-----. 1 oracle oinstall 262152192 Apr  6 14:07 system01.dbf
[oracle@bernardo cdbora]$ ls -l testepdb/
total 860240
-rw-r-----. 1 oracle oinstall  20979712 Apr  7 18:50 pdbseed_temp01.dbf
-rw-r-----. 1 oracle oinstall 618668032 Apr  7 18:50 sysaux01.dbf
-rw-r-----. 1 oracle oinstall 262152192 Apr  7 18:50 system01.dbf

{% endhighlight %}

Podem notar que até o nome do datafile de temp foi copiado, caso queira, 
pode-se modificar manualmente.

{% highlight sql %}

set lines 200
col file_name format a60
col tablespace_name format a10
select a.file_name,a.tablespace_name,con_id from cdb_temp_files a;

FILE_NAME						     TABLESPACE     CON_ID
------------------------------------------------------------ ---------- ----------
/u01/app/oracle/oradata/cdbora/pdbseed/pdbseed_temp01.dbf    TEMP		 2
/u01/app/oracle/oradata/cdbora/temp01.dbf		     TEMP		 1
/u01/app/oracle/oradata/cdbora/testepdb/testepdb_temp01.dbf  TEMP		 4
/u01/app/oracle/oradata/cdbora/pdbora/pdbora_temp01.dbf      TEMP		 3

--Logue no pdb e modifique os temp files

alter session set container=testepdb

Session altered.

alter tablespace temp add 
tempfile '/u01/app/oracle/oradata/cdbora/testepdb/testepdb_temp01.dbf' size 20m;

Tablespace altered.

alter database drop 
tempfile '/u01/app/oracle/oradata/cdbora/testepdb/pdbseed_temp01.dbf';

Tablespace altered.

{% endhighlight %}


Observe através da query que após a criação ainda devemos inciar o PDB.

{% highlight sql %}
select con_id, name, open_mode, restricted from v$pdbs order by 1;

    CON_ID NAME 			  OPEN_MODE  RES
---------- ------------------------------ ---------- ---
	 2 PDB$SEED			  READ ONLY  NO
	 3 PDBORA			  READ WRITE NO
	 4 TESTEPDB			  MOUNTED

--Use o comando abaixo para iniciar o PDB

alter pluggable database testepdb open;

Pluggable database altered.

select con_id, name, open_mode, restricted from v$pdbs order by 1;

    CON_ID NAME 			  OPEN_MODE  RES
---------- ------------------------------ ---------- ---
	 2 PDB$SEED			  READ ONLY  NO
	 3 PDBORA			  READ WRITE NO
	 4 TESTEPDB			  READ WRITE NO

{% endhighlight %}

Conexão com o novo PDB

{% highlight bash %}

--tnsnames.ora
TESTEPDB =
  (DESCRIPTION =
    (ADDRESS = (PROTOCOL = TCP)(HOST = seu.host.com)(PORT = 1522))
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = testepdb)
    )
  )

[oracle@bernardo cdbora]$ sqlplus testeadm/testeadm@testepdb

SQL*Plus: Release 12.1.0.1.0 Production on Mon Apr 7 20:09:44 2014

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Last Successful login time: Mon Apr 07 2014 20:08:32 -03:00

Connected to:
Oracle Database 12c Enterprise Edition Release 12.1.0.1.0 - 64bit Production
With the Partitioning, OLAP, Advanced Analytics and Real Application Testing options

SQL> show con_name    

CON_NAME
------------------------------
TESTEPDB
{% endhighlight %}

#Conclusão

Está é a abordagem mais prática de adição de um novo PDB quando existe a necessidade de criar a base do zero, bem mais fácil que na arquitetura antiga a qual teríamos que criar um novo banco de dados.

Também pode-se usar o DBCA para fazer o mesmo esquema para aqueles que preferem uma interface gráfica.