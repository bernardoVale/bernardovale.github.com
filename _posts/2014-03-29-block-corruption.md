---
layout: post
title: Block Media Recovery Lab
description: "Forçando um block corruption e operando block media recovery"
tags : [recovery, oracle, tutorial, ORA-01578, ORA-01110, block ]
image:
  background: block.jpg
comments: true
share: true
---

# Simulando um Block Recovery

## Objetivo

Devido a remota chance de ocorrer alguma falha a ponto de um recovery ser a solução, muitas vezes um DBA precisa criar simulações para garantir que quando o problema acontecer esteja apto a solucioná-lo.

A recuperação a nível de bloco não é uma exceção, dificilmente teremos que fazer recover a esse nível.

> Block recovery só está disponível na versão Enterprise.

## Montando o Ambiente

Vamos dividir este trabalho em duas partes. A primeira, gerar um block corruption, segunda, recuperando o bloco.

### Gerando um block recovery
* Criar uma tabela de simulação com os registros da `dba_tables`
{% highlight sql %}
create table blog.block_corruption as select * from dba_tables;

Table created.
{% endhighlight %}
* Identificar o(s) segmento(s) da tabela criada
{% highlight sql %}
col segment_name format a20;
select segment_name, tablespace_name from dba_segments where segment_name='BLOCK_corruption';

SEGMENT_NAME         TABLESPACE_NAME
-------------------- ------------------------------
BLOCK_corruption     BLOG

{% endhighlight %}

* Identificar através do segmento, o bloco correspondente ao seu cabeçalho.
{% highlight sql %}
col name format a50; 
select s.header_file, s.header_block, d.name from dba_segments s 
inner join v$datafile d on (s.header_file = d.file#) where segment_name='BLOCK_corruption';SQL>   2  

HEADER_FILE HEADER_BLOCK NAME
----------- ------------ -------------------------------------------
         41          130 /u01/app/oracle/oradata/dbconn/blog01.dbf
{% endhighlight %}
* Modificar o cabeçalho do segmento pelo Sistema Operacional com a ferramenta dd
{% highlight bash %}
# File= Datafile que iremos modificar
# bs= tamanho do bloco desse datafile. Cheque o parametro db_block_size
# notrunc=true significa que iremos substituir os dados do bloco
# seek=NUM  numero do bloco que valos modificas

# Mude o bloco correspondente ao header +1 provavelmente já será um bloco de dados da tabela

[oracle@bernardo ~]$ dd of=/u01/app/oracle/oradata/dbconn/blog01.dbf bs=8192 conv=notrunc seek=131 << EOF
> 'Em verdade vos digo, esse bloco está corrompido'
> EOF
0+1 records in
0+1 records out
51 bytes (51 B) copied, 3.882e-05 s, 1.3 MB/s

# Mais um bloquinho

dd of=/u01/app/oracle/oradata/dbconn/users01.dbf bs=8192 conv=notrunc seek=828 << EOF
adssdaakjsidsaashiahadh
EOF
0+1 records in
0+1 records out
24 bytes (24 B) copied, 4.4148e-05 s, 544 kB/s
{% endhighlight %}

Para descobrir se a operação correu bem, podemos identificar a falha de diversas formas, segue abaixo:

* Executar uma query que precisa dos blocos corrompidos
{% highlight sql %}

SQL> select table_name from blog.block_corruption;
select table_name from blog.block_corruption
                            *
ERROR at line 1:
ORA-01578: ORACLE data block corrupted (file # 41, block # 131)
ORA-01110: data file 41: '/u01/app/oracle/oradata/dbconn/blog01.dbf'
{% endhighlight %}

* Usar o utilitário dbv
{% highlight bash %}
[oracle@bernardo ~]$ dbv file=/u01/app/oracle/oradata/dbconn/blog01.dbf blocksize=8192

DBVERIFY: Release 11.2.0.3.0 - Production on Sun Mar 30 00:06:32 2014

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

DBVERIFY - Verification starting : FILE = /u01/app/oracle/oradata/dbconn/blog01.dbf
Page 131 is marked corrupt
Corrupt block relative dba: 0x0a400083 (file 41, block 131)
Bad header found during dbv: 
Data in bad block:
 type: 39 format: 5 rdba: 0x64726576
 last change scn: 0x6f76.20656461 seq: 0x73 flg: 0x20
 spare1: 0x6d spare2: 0x20 spare3: 0x6f67
 consistency value in tail: 0x5c6c0601
 check value in block header: 0x6964
 block checksum disabled



DBVERIFY - Verification complete

Total Pages Examined         : 12800
Total Pages Processed (Data) : 96
Total Pages Failing   (Data) : 0
Total Pages Processed (Index): 0
Total Pages Failing   (Index): 0
Total Pages Processed (Other): 136
Total Pages Processed (Seg)  : 0
Total Pages Failing   (Seg)  : 0
Total Pages Empty            : 12567
Total Pages Marked Corrupt   : 1
Total Pages Influx           : 0
Total Pages Encrypted        : 0
Highest block SCN            : 76438665 (0.76438665)

{% endhighlight %}

* Executando qualquer tipo de ANALYZE TABLE

{% highlight sql %}
SQL> analyze table blog.block_corruption compute statistics;
analyze table blog.block_corruption compute statistics
*
ERROR at line 1:
ORA-01578: ORACLE data block corrupted (file # 41, block # 131)
ORA-01110: data file 41: '/u01/app/oracle/oradata/dbconn/blog01.dbf'
{% endhighlight %}

* Validando o datafile pelo rman

{% highlight bash %}

RMAN> validate datafile 41;

Starting validate at 30-MAR-14
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=25 device type=DISK
allocated channel: ORA_DISK_2
channel ORA_DISK_2: SID=148 device type=DISK
allocated channel: ORA_DISK_3
channel ORA_DISK_3: SID=21 device type=DISK
allocated channel: ORA_DISK_4
channel ORA_DISK_4: SID=139 device type=DISK
channel ORA_DISK_1: starting validation of datafile
channel ORA_DISK_1: specifying datafile(s) for validation
input datafile file number=00041 name=/u01/app/oracle/oradata/dbconn/blog01.dbf
channel ORA_DISK_1: validation complete, elapsed time: 00:00:01
List of Datafiles
=================
File Status Marked Corrupt Empty Blocks Blocks Examined High SCN
---- ------ -------------- ------------ --------------- ----------
41   FAILED 0              12567        12800           76440591  
  File Name: /u01/app/oracle/oradata/dbconn/blog01.dbf
  Block Type Blocks Failing Blocks Processed
  ---------- -------------- ----------------
  Data       0              95              
  Index      0              0               
  Other      2              138 
{% endhighlight %}

## Recuperando o bloco

Use a `v$database_block_corruption` para checar todos os blocos corrompidos
que o Oracle encontrou.

{% highlight sql %}
SQL> select * from v$database_block_corruption;

     FILE#     BLOCK#     BLOCKS CORRUPTION_CHANGE# CORRUPTIO
---------- ---------- ---------- ------------------ ---------
        41        131          1                  0 CORRUPT
{% endhighlight %}

Vale salientar que para recuperar esse bloco precisaremos de backup full ou então de flashback logs disponíveis para garantir que o oracle conseguirá obter uma cópia integra desse bloco e realizar a operação de recovery.

> Verifique o valor do parametro [db_block_checking](http://docs.oracle.com/cd/B19306_01/server.102/b14237/initparams039.htm) para garantir que o Oracle está checando esse tipo de corrupção.

O recovery pode ser feito bloco a bloco ou na totalidade usando todas as entradas da view `v$database_block_corruption`.

* Por bloco

{% highlight bash %}

RMAN> recover datafile 41 block 131;

Starting recover at 30-MAR-14
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=139 device type=DISK
allocated channel: ORA_DISK_2
channel ORA_DISK_2: SID=23 device type=DISK
allocated channel: ORA_DISK_3
channel ORA_DISK_3: SID=142 device type=DISK
allocated channel: ORA_DISK_4
channel ORA_DISK_4: SID=25 device type=DISK

channel ORA_DISK_1: restoring block(s)
channel ORA_DISK_1: specifying block(s) to restore from backup set
restoring blocks of datafile 00041
channel ORA_DISK_1: reading from backup piece /u01/app/oracle/product/11.2.0/db_1/dbs/0hp4eb0l_1_1
channel ORA_DISK_1: piece handle=/u01/app/oracle/product/11.2.0/db_1/dbs/0hp4eb0l_1_1 tag=TAG20140330T003221
channel ORA_DISK_1: restored block(s) from backup piece 1
channel ORA_DISK_1: block restore complete, elapsed time: 00:00:01

starting media recovery
media recovery complete, elapsed time: 00:00:01

Finished recover at 30-MAR-14
{% endhighlight %}

* Todas as entradas

{% highlight bash %}

RMAN> recover corruption list;

Starting recover at 30-MAR-14
using target database control file instead of recovery catalog
allocated channel: ORA_DISK_1
channel ORA_DISK_1: SID=142 device type=DISK
allocated channel: ORA_DISK_2
channel ORA_DISK_2: SID=21 device type=DISK
allocated channel: ORA_DISK_3
channel ORA_DISK_3: SID=146 device type=DISK
allocated channel: ORA_DISK_4
channel ORA_DISK_4: SID=23 device type=DISK

channel ORA_DISK_1: restoring block(s)
channel ORA_DISK_1: specifying block(s) to restore from backup set
restoring blocks of datafile 00041
channel ORA_DISK_1: reading from backup piece /u01/app/oracle/product/11.2.0/db_1/dbs/0hp4eb0l_1_1
channel ORA_DISK_1: piece handle=/u01/app/oracle/product/11.2.0/db_1/dbs/0hp4eb0l_1_1 tag=TAG20140330T003221
channel ORA_DISK_1: restored block(s) from backup piece 1
channel ORA_DISK_1: block restore complete, elapsed time: 00:00:01

starting media recovery
media recovery complete, elapsed time: 00:00:03

Finished recover at 30-MAR-14

{% endhighlight %}

## Bonus Script

Fiz um script para gerar o comando o dd mais facilmente. É só usar as variáveis SCHEMA e TABLE
{% highlight sql %}
set heading off
set lines 113
SELECT 'dd of=' || f.file_name || ' bs=8192 conv=notrunc seek=' ||
       to_number(S.HEADER_BLOCK + 1) || ' << EOF',
       'EM VERDADE EU VOS DIGO, DEPOIS QUE SORTA ESSE COMANDO ESSE BLOCO TA CORROMPIDO',
       'EOF'
  FROM DBA_SEGMENTS S, dba_data_files f, dba_tables t
 WHERE f.tablespace_name = t.tablespace_name
   and S.SEGMENT_NAME = t.table_name
   and t.table_name = '&TABLE'
   and S.OWNER = t.owner
   and t.owner = '&SCHEMA';
{% endhighlight %}