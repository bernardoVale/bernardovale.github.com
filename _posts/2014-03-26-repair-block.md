---
layout: post
title: Nologging Operations
description: "Recuperando uma tabela após um DML nologging no Oracle"
tags : [recovery, oracle, tutorial, ORA-26040, ORA-01110, nologging ]
comments: true
share: true
---

## Cenário

Foram geradas algumas transações com a opção NOLOGGING, não foi feito um backup após a operação e por castigo divino perdemos o datafile correspondente.

### Montagem do cenário exemplo

Vamos criar uma tabela exemplo e inserir alguns dados com a cláusula `NOLOGGING`.
{% highlight sql %}
    SQL> create table scott.corruption as select * from dba_tables;

    Table created.
    
    SQL> alter table scott.corruption nologging;

    Table altered.    

    SQL> insert /*+ APPEND */ into scott.corruption (select * from all_tables);

    2749 rows created.

    SQL> commit;

    Commit complete.
    
    SQL> select count(*) from scott.corruption;

    COUNT(*)
    --------
      5497
{% endhighlight %}
Antes e depois de qualquer operação `nologging` é recomendado um backup. Podemos confirmar o êxito da operação através do `RMAN`.

{% highlight bash %}
    RMAN> report unrecoverable;

    using target database control file instead of recovery catalog
    Report of files that need backup due to unrecoverable operations
    File Type of Backup Required Name
    ------------------------------------------------------------------------
    4    full or incremental     /u01/app/oracle/oradata/dbconn/users01.dbf 
{% endhighlight %}

## Corrompendo o Bloco

Com a finalidade de simular o nosso caso de estudo, iremos remover o datafile correspondente e faremos um `datafile recover`.
{% highlight bash %}
     RMAN> host;
     [oracle@bernardo datapump]$ rm /u01/app/oracle/oradata/dbconn/users01.dbf
     [oracle@bernardo datapump]$ exit
     exit
     host command complete
     RMAN> run {
     sql 'alter database datafile 4 offline';
     restore datafile 4;
     recover datafile 4;
     sql 'alter database datafile 4 online';
     }
     
     using target database control file instead of recovery catalog
     sql statement: alter database datafile 4 offline
     
     Starting restore at 26-MAR-14
     allocated channel: ORA_DISK_1
     channel ORA_DISK_1: SID=152 device type=DISK
     allocated channel: ORA_DISK_2
     channel ORA_DISK_2: SID=27 device type=DISK
     allocated channel: ORA_DISK_3
     channel ORA_DISK_3: SID=21 device type=DISK
     allocated channel: ORA_DISK_4
     channel ORA_DISK_4: SID=146 device type=DISK
     
     channel ORA_DISK_1: starting datafile backup set restore
     channel ORA_DISK_1: specifying datafile(s) to restore from backup set
     channel ORA_DISK_1: restoring datafile 00004 to /u01/app/oracle/oradata/dbconn/users01.dbf
     channel ORA_DISK_1: reading from backup piece /u01/app/oracle/product/11.2.0/db_1/dbs/0ap4510t_1_1
     channel ORA_DISK_1: piece handle=/u01/app/oracle/product/11.2.0/db_1/dbs/0ap4510t_1_1 tag=TAG20140326T114636
     channel ORA_DISK_1: restored backup piece 1
     channel ORA_DISK_1: restore complete, elapsed time: 00:00:01
     Finished restore at 26-MAR-14
     
     Starting recover at 26-MAR-14
     using channel ORA_DISK_1
     using channel ORA_DISK_2
     using channel ORA_DISK_3
     using channel ORA_DISK_4
     
     starting media recovery
     media recovery complete, elapsed time: 00:00:00
     
     Finished recover at 26-MAR-14
     
     sql statement: alter database datafile 4 online
{% endhighlight %}

Uma simples query que leria todos os blocos já nos traria um erro.
{% highlight sql %}

    SQL> select count(*) from scott.corruption;
    select count(*) from scott.corruption
                           *
    ERROR at line 1:
    ORA-01578: ORACLE data block corrupted (file # 4, block # 802)
    ORA-01110: data file 4: '/u01/app/oracle/oradata/dbconn/users01.dbf'
    ORA-26040: Data block was loaded using the NOLOGGING option
{% endhighlight %}
##Ação Corretiva.

Na verdade, não há muito o que fazer nessa situação, como a tabela estava no modo `nologging` não foi gerado redo, sendo assim, não existe um meio de recuperar esses dados nesse cenário. O DBA deverá marcar os blocos como corrompidos e permitir DML novamente na tabela através de um script da Oracle indicado no ID 556733.1.

{% highlight sql %}
    SQL> REM Identify corrupted blocks for schema.object:
    set serveroutput on
    DECLARE num_corrupt INT;
    BEGIN
      num_corrupt := 0;
      DBMS_REPAIR.CHECK_OBJECT (
      SCHEMA_NAME => '&schema_name',
      OBJECT_NAME => '&object_name',
      REPAIR_TABLE_NAME => 'REPAIR_TABLE',
      corrupt_count => num_corrupt);
      DBMS_OUTPUT.PUT_LINE('number corrupt: ' || TO_CHAR (num_corrupt));
    END;
    /SQL> SQL> SQL>   2    3    4    5    6    7    8    9   10   11  
    Enter value for schema_name: SCOTT
    old   5:   SCHEMA_NAME => '&schema_name',
    new   5:   SCHEMA_NAME => 'SCOTT',
    Enter value for object_name: CORRUPTION
    old   6:   OBJECT_NAME => '&object_name',
    new   6:   OBJECT_NAME => 'CORRUPTION',
    number corrupt: 97
    
    PL/SQL procedure successfully completed.
    SQL> REM Allow future DML statements to skip the corrupted blocks:
    
    BEGIN
      DBMS_REPAIR.SKIP_CORRUPT_BLOCKS (
      SCHEMA_NAME => '&schema_name',
      OBJECT_NAME => '&object_name',
      OBJECT_TYPE => dbms_repair.table_object,
      FLAGS => dbms_repair.SKIP_FLAG);
    END;
    /SQL> SQL>   2    3    4    5    6    7    8  
    Enter value for schema_name: SCOTT
    old   3:   SCHEMA_NAME => '&schema_name',
    new   3:   SCHEMA_NAME => 'SCOTT',
    Enter value for object_name: CORRUPTION
    old   4:   OBJECT_NAME => '&object_name',
    new   4:   OBJECT_NAME => 'CORRUPTION',
    PL/SQL procedure successfully completed.
{% endhighlight %}

Após esse procedimento podemos fazer a mesma query, note que os blocos corrompidos foram ignorados e não existem mais.    
 {% highlight sql %}   
    SQL> select count(*) from scott.corruption;
    
      COUNT(*)	
       --------
       2748
{% endhighlight %}
#Conclusão

Esse tipo de operação deve ser evitada, é altamente recomendável que nunca a façamos, seguindo esse exemplo, fica claro como essa ação pode ser perigosa.