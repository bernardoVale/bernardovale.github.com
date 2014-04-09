---
layout: post
title: Datafile Recovery via LSOF
description: "Salvando vidas após um rm -rf"
tags : [recovery, lsof, ORA-01110, ORA-01116, ORA-27041, ORA-03113 ]
comments: true
share: true
---

## Cenário

Algum DBA/SYSADMIN juvenil removeu um datafile via SO, porém, todos os usuários que usam aquele datafile continuam trabalhando normalmente (Os dados ainda estão nos redo logs).

## O que é o LSOF?

O comando `lsof` é utilizado para mostrar os arquivos que estão abertos no sistema, portanto, podemos usa-lo para checar os datafiles que o DBWriter está segurando e usa-lo como parâmetro para recriar o datafile perdido.

##Montando o Ambiente

Para testar esse método de recovery criaremos uma tablespace, um usuário e uma tabela, depois vamos inserir dados e remover o datafile.

{% highlight sql %}

--Criando uma tablespace
create tablespace RECOV datafile '/u01/app/oracle/oradata/oradb/recov01.dbf' size 10m;

Tablespace created.

create user recov identified by recov default tablespace recov;

User created.

grant connect,resource to recov;

Grant succeeded.

--Criar uma tabela dentro dessa tablespace

create table recov.test_recov (data timestamp);

Table created.

-- Inserir dados no cara

conn recov/recov

Connected.

insert into test_recov values (systimestamp);

1 row created.

insert into test_recov values (systimestamp);

1 row created.

insert into test_recov values (systimestamp);

1 row created.

insert into test_recov values (systimestamp);

1 row created.

commit;

Commit complete.

select * from test_recov;

DATA
-------------------------
09/04/14 14:06:58,379547
09/04/14 14:07:01,450273
09/04/14 14:07:02,423897
09/04/14 14:07:03,636331

{% endhighlight %}

## ATENÇÃO

Nesse modo de recovery, devemos agir rapidamente, pois, o DBwriter pode perder a referência a esse arquivo e então teremos que colocar o datafile offline e fazer o bom e velho restore/recover.

**Não ponha o datafile offline**

**Não de um checkpoint** 


Removeremos o datafile e simularemos a continuidade do trabalho adicionado mais registros.

{% highlight bash %}

rm /u01/app/oracle/oradata/oradb/recov01.dbf

{% endhighlight %}

{% highlight sql %}

conn recov/recov
Connected.

insert into test_recov values (systimestamp);

1 row created.

commit;

Commit complete.

select * from test_recov;

DATA
-------------------------
09/04/14 14:06:58,379547
09/04/14 14:07:01,450273
09/04/14 14:07:02,423897
09/04/14 14:07:03,636331
09/04/14 14:07:54,627772

{% endhighlight %}

Observando o alertlog veremos que o Oracle já deu falta do datafile.

{% highlight bash %}

ORA-01116: error in opening database file 14
ORA-01110: data file 14: '/u01/app/oracle/oradata/oradb/recov01.dbf'
ORA-27041: unable to open file
Linux-x86_64 Error: 2: No such file or directory
Additional information: 3

{% endhighlight %}

## LSOF em Ação

Primeiro devemos achar o processo do DBwriter
{% highlight bash %}

[oracle@bernardo trace]$ ps -ef | grep dbw
oracle    7840 30923  0 14:08 pts/0    00:00:00 grep dbw
oracle   32096     1  0 12:02 ?        00:00:15 ora_dbw0_oradb

{% endhighlight %}

Usando o número do processo podemos checar se ele ainda tem a referência ao datafile
{% highlight bash %}
[oracle@bernardo trace]$ lsof -p 32096 | grep .dbf
oracle  32096 oracle  258uW  REG    252,0  10493952   544090 /u01/app/oracle/oradata/oradb/rmrf_rec01.dbf
oracle  32096 oracle  259uW  REG    252,0 891297792   538287 /u01/app/oracle/oradata/oradb/system01.dbf
oracle  32096 oracle  260uW  REG    252,0 922755072   538445 /u01/app/oracle/oradata/oradb/sysaux01.dbf
oracle  32096 oracle  261uW  REG    252,0 346038272   538446 /u01/app/oracle/oradata/oradb/undotbs01.dbf
oracle  32096 oracle  262uW  REG    252,0 209723392   538447 /u01/app/oracle/oradata/oradb/users01.dbf
oracle  32096 oracle  263uW  REG    252,0 362422272   538455 /u01/app/oracle/oradata/oradb/example01.dbf
oracle  32096 oracle  264uW  REG    252,0 536879104   538458 /u01/app/oracle/oradata/oradb/tsd_sistema.dbf
oracle  32096 oracle  265uW  REG    252,0 536879104   538459 /u01/app/oracle/oradata/oradb/tsI_sistema.dbf
oracle  32096 oracle  266uW  REG    252,0 402661376   539381 /u01/app/oracle/oradata/oradb/oggddl_01.dbf
oracle  32096 oracle  267uW  REG    252,0 134225920   540355 /u01/app/oracle/oradata/oradb/oggdba_01.dbf
oracle  32096 oracle  268uW  REG    252,0  24125440   543974 /u01/app/oracle/oradata/rman_catalog_01.dbf
oracle  32096 oracle  269uW  REG    252,0  31465472   543973 /u01/app/oracle/oradata/oradb/ts_users_isse01.dbf
oracle  32096 oracle  270uW  REG    252,0  52436992   544032 /u01/app/oracle/oradata/oradb/trans_01.dbf
oracle  32096 oracle  271uW  REG    252,0 268443648   538454 /u01/app/oracle/oradata/oradb/temp01.dbf
oracle  32096 oracle  272uW  REG    252,0  10493952   544027 /u01/app/oracle/oradata/oradb/recov01.dbf (deleted)
{% endhighlight %}

Observem que o arquivo esta marcado como `deleted` porém ainda podemos recuperá-lo

Na última linha está o arquivo `272` que aponta para o datafile deletado.
{% highlight bash %}
ls -ltar /proc/32096/fd
[oracle@bernardo trace]$ ls -ltar /proc/32096/fd
total 0
dr-xr-xr-x. 8 oracle oinstall  0 Apr  9 12:02 ..
dr-x------. 2 oracle oinstall  0 Apr  9 12:02 .
lr-x------. 1 oracle oinstall 64 Apr  9 12:09 9 -> /dev/null
lr-x------. 1 oracle oinstall 64 Apr  9 12:09 8 -> /dev/null
lr-x------. 1 oracle oinstall 64 Apr  9 12:09 7 -> /dev/null
lr-x------. 1 oracle oinstall 64 Apr  9 12:09 6 -> /dev/null
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 5 -> /u01/app/oracle/product/11.2.0/db_1/dbs/hc_oradb.dat
lr-x------. 1 oracle oinstall 64 Apr  9 12:09 4 -> /dev/null
lr-x------. 1 oracle oinstall 64 Apr  9 12:09 3 -> /dev/null
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 271 -> /u01/app/oracle/oradata/oradb/temp01.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 270 -> /u01/app/oracle/oradata/oradb/trans_01.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 269 -> /u01/app/oracle/oradata/oradb/ts_users_isse01.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 268 -> /u01/app/oracle/oradata/rman_catalog_01.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 267 -> /u01/app/oracle/oradata/oradb/oggdba_01.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 266 -> /u01/app/oracle/oradata/oradb/oggddl_01.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 265 -> /u01/app/oracle/oradata/oradb/tsI_sistema.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 264 -> /u01/app/oracle/oradata/oradb/tsd_sistema.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 263 -> /u01/app/oracle/oradata/oradb/example01.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 262 -> /u01/app/oracle/oradata/oradb/users01.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 261 -> /u01/app/oracle/oradata/oradb/undotbs01.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 260 -> /u01/app/oracle/oradata/oradb/sysaux01.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 259 -> /u01/app/oracle/oradata/oradb/system01.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 258 -> /u01/app/oracle/oradata/oradb/rmrf_rec01.dbf
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 257 -> /u01/app/oracle/oradata/oradb/control02.ctl
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 256 -> /u01/app/oracle/oradata/oradb/control01.ctl
lr-x------. 1 oracle oinstall 64 Apr  9 12:09 21 -> /u01/app/oracle/product/11.2.0/db_1/rdbms/mesg/oraus.msb
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 20 -> socket:[45063380]
l-wx------. 1 oracle oinstall 64 Apr  9 12:09 2 -> /dev/null
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 17 -> /u01/app/oracle/product/11.2.0/db_1/dbs/lkORADB
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 16 -> /u01/app/oracle/product/11.2.0/db_1/dbs/hc_oradb.dat
lr-x------. 1 oracle oinstall 64 Apr  9 12:09 15 -> /dev/zero
lr-x------. 1 oracle oinstall 64 Apr  9 12:09 14 -> /proc/32096/fd
lr-x------. 1 oracle oinstall 64 Apr  9 12:09 13 -> /u01/app/oracle/product/11.2.0/db_1/rdbms/mesg/oraus.msb
lrwx------. 1 oracle oinstall 64 Apr  9 12:09 12 -> /u01/app/oracle/product/11.2.0/db_1/dbs/hc_oradb.dat
lr-x------. 1 oracle oinstall 64 Apr  9 12:09 11 -> /dev/zero
lr-x------. 1 oracle oinstall 64 Apr  9 12:09 10 -> /dev/zero
l-wx------. 1 oracle oinstall 64 Apr  9 12:09 1 -> /dev/null
lr-x------. 1 oracle oinstall 64 Apr  9 12:09 0 -> /dev/null
lrwx------. 1 oracle oinstall 64 Apr  9 14:03 272 -> /u01/app/oracle/oradata/oradb/recov01.dbf (deleted)
{% endhighlight %}

O resto já podem imaginar, é só dar um cat no arquivo e apontar para o seu nome antigo

{% highlight bash %}
[oracle@bernardo trace]$ cat /proc/32096/fd/272 > /u01/app/oracle/oradata/oradb/recov01.dbf
{% endhighlight %}

Após o comando tente inserir mais dados e fazer um checkpoint, se obtiver sucesso nessa operação o recovery foi um sucesso, caso contrário, faça o restore do seu último backup full.

{% highlight sql %}

insert into recov.test_recov values (systimestamp);

1 row created.

commit;

Commit complete.

alter system checkpoint;

System altered.

select * from recov.test_recov;

DATA
--------------------------
09/04/14 14:10:56,806180
09/04/14 14:06:58,379547
09/04/14 14:07:01,450273
09/04/14 14:07:02,423897
09/04/14 14:07:03,636331
09/04/14 14:07:54,627772

6 rows selected.

{% endhighlight %}

##Conclusão

**Antes de fazer cagada, use o `lsof`, talvez não seja necessário colocar o datafile no modo offline.**
