---
layout: post
title: Monitorando os Índices
description: "Monitorando o uso de índices"
tags : [oracle, tunning, index, ORA-25176, ORA-22864]
comments: true
share: true
---

# Objetivo

É muito comum encontrarmos em diversos ambientes uma grande quantidade de índices que estão em total desuso. Não adianta sair criando índices igual um louco, deve-se estudar a real necessidade do mesmo.

Índices são excelentes objetos para otimizar o acesso aos dados, principalmente em tabelas com pouca modificação e muita pesquisa. Isso todo mundo sabe, porém, as vezes esquecemos que índices também podem ser causadores de problemas de performance, ora, toda vez que a tabela for modificada, todos os índices que a referenciam, terão que ser modificados também.

A proposta desse artigo é demonstrar como podemos monitorar o uso dos índices. Será que podemos remover este índices? Está sendo usado? Quando foi usado?

> É recomendável que o monitoramento de índices seja uma tarefa rotineira de um DBA.

## Monitoring Usage clause

O oracle permite através da cláusula `monitoring usage` o registro de logs quando o índice for utilizado.

No nosso estudo de caso, iremos usar o schema HR (Human Resources) do pacotes de usuários de exemplo do Oracle.

{% highlight sql %}
--Ativar
alter index DEPT_ID_PK monitoring usage;

Index altered. 
--Desativar
alter index DEPT_ID_PK nomonitoring usage;

Index altered.
{% endhighlight %}

Após ativar o monitoramento, quando o índice for usado uma entrada na view `v$object_usage` será feita. Faremos uma query que usa o índice para testar o monitoramento.

{% highlight sql %}

set autotrace on;
select employee_id from hr.employees where employee_id = 102;
EMPLOYEE_ID
-----------
	102


Execution Plan
----------------------------------------------------------
Plan hash value: 1386472110

-----------------------------------------------------------------------------------
| Id  | Operation	  | Name	  | Rows  | Bytes | Cost (%CPU)| Time	  |
-----------------------------------------------------------------------------------
|   0 | SELECT STATEMENT  |		  |	1 |	4 |	0   (0)| 00:00:01 |
|*  1 |  INDEX UNIQUE SCAN| EMP_EMP_ID_PK |	1 |	4 |	0   (0)| 00:00:01 |
-----------------------------------------------------------------------------------

set autotrace off;
select * from v$object_usage;

INDEX_NAME		      |TABLE_NAME  |MON|USE|START_MONITORING   |END_MONITORING
------------------------------|------------|---|---|-------------------|---------------
EMP_EMP_ID_PK		      |EMPLOYEE    |YES|YES|04/18/2014 22:54:30|

{% endhighlight %}

###Observações

* A view `v$object_usage` grava os dados somente do usuário que fez a query.
* A view guarda apenas um registro por índice
* A coluna USE significa que o índice foi utilizado
* A coluna Monitoring indica que o índice está no modo de monitoramento 
* Não é possível monitorar índice de tabela IOT(Index organized table), erro ORA-25176
* Não é possível monitorar um índice LOB, erro ORA-22864.

## Monitorando todos os usuários

Para monitorar a todos, podemos usar a query usada para criar a view `v$object_usage` como base para uma nova view, sem a restrição do usuário. 

{% highlight sql %}

set long 1500
select text from dba_views where view_name = 'V$OBJECT_USAGE';

TEXT
--------------------------------------------------------------
select io.name, t.name,
       decode(bitand(i.flags, 65536), 0, 'NO', 'YES'),
       decode(bitand(ou.flags, 1), 0, 'NO', 'YES'),
       ou.start_monitoring,
       ou.end_monitoring
from sys.obj$ io, sys.obj$ t, sys.ind$ i, sys.object_usage ou
where io.owner# = userenv('SCHEMAID')
  and i.obj# = ou.obj#
  and io.obj# = ou.obj#
  and t.obj# = i.bo#

--VAMOS CRIAR UMA VIEW SEM A LINHA USERENV('SCHEMAID')
--Adicionei a tabela dba_objects para ter o owner e 
--possivelmente filtrar por usuário

CREATE VIEW dba_object_usage (owner,
index_name, table_name, monitoring, used, start_monitoring, end_monitoring )
AS
SELECT do.owner, io.NAME, t.NAME,
DECODE (BITAND (i.flags, 65536), 0, 'NO', 'YES'),
DECODE (BITAND (ou.flags, 1), 0, 'NO', 'YES'), ou.start_monitoring,
ou.end_monitoring
FROM SYS.obj$ io,
SYS.obj$ t,
SYS.ind$ i,
SYS.object_usage ou,
dba_objects do
WHERE i.obj# = ou.obj#
AND io.obj# = ou.obj#
AND t.obj# = i.bo#
AND ou.obj# = do.object_id;

View created.

{% endhighlight %}

Após a criação da view podemos fazer uma query restringindo o usuário e verificar todos os índices que foram usados desde que ativamos o monitoramento.

{% highlight sql %}

select index_name 
from dba_object_usage
where owner = 'HR'
and used = 'YES';

INDEX_NAME
---------------
EMP_EMP_ID_PK

{% endhighlight %}

## Bonus Script

Use os scripts abaixo para alterar todos os índices do usuário especificado.

{% highlight sql %}
-- -----------------------------------------------------------------------------------
-- File Name    : index_monitoring_on.sql
-- Author       : Bernardo Vale
-- Description  : Ativa o monitoramento de todos os indices do usuário
-- Call Syntax  : @index_monitoring_on (USUARIO)
-- Last Modified: 18/04/2014
-- -----------------------------------------------------------------------------------
BEGIN
  FOR c IN
  ( select c.owner, c.index_name, c.index_type
	from dba_indexes c
	where c.owner='&OWNER'
	and c.index_type not in ('IOT - TOP','LOB'))
  LOOP
    dbms_utility.exec_ddl_statement('alter index ' || c.owner || '.' 
      || c.index_name || ' monitoring usage');
  end loop;
end;

-- -----------------------------------------------------------------------------------
-- File Name    : index_monitoring_off.sql
-- Author       : Bernardo Vale
-- Description  : Desativa o monitoramento de todos os indices do usuário
-- Call Syntax  : @index_monitoring_on (USUARIO)
-- Last Modified: 18/04/2014
-- -----------------------------------------------------------------------------------
BEGIN
  FOR c IN
  ( select c.owner, c.index_name, c.index_type
	from dba_indexes c
	where c.owner='HR'
	and c.index_type not in ('IOT - TOP','LOB'))
  LOOP
    dbms_utility.exec_ddl_statement('alter index ' || c.owner || '.' 
     || c.index_name || ' nomonitoring usage');
  end loop;
end;

{% endhighlight %}

#Outros métodos de monitoramento

O monitoramento dos índices pode ser feito de outras maneiras. O evangelista Oracle Damir Vadas escreveu um excelente artigo sobre esse assunto. Segundo Damir, pelo fato de se gerar apenas um registro por índice, este método pode não ser o mais correto para o monitoramento dos índices.

> O método utilizado por Damir requer as options do Oracle Enterprise,  Tunning pack e Diagnostic pack

[Link do artigo](http://damir-vadas.blogspot.com.br/2010/11/how-to-see-index-usage-without-alter.html)
