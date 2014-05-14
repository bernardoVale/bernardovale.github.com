---
layout: post
title: Evento SQL*Net Message From Client
description: "Descruba o significa do evento de wait SQL*Net Message from client"
tags : [tunning, wait, idle, sql*net]
comments: true
share: true
---

## Tunning Eventos Idle

Estava eu envolvido um projeto de tunning muito particular, todas as query's que o cliente repassava estavam executando muito rápido. Achei muito estranho as reclamações e pedi um trace que incluisse os eventos de wait.

Descobri então que a maior parte do tempo da execução dos processos eram perdidas no processamento local.

## Exemplificando

Para demonstrar como um evento deste é capturado pelo Oracle criei um test-case utilizando uma aplicação em Python que se conecta no banco e a cada iteração fica cinco segundos parada (simulando um tipo de processamento que o cliente poderia estar fazendo). Em seguida coletei o trace da sessão e analisei o resultado.

# A Aplicação

{% highlight python %}
#!/usr/bin/python
__author__ = 'Bernardo Vale'
__enterprise__ = 'LB2 Consultoria e Tecnologia LTDA'
import time
import cx_Oracle

con = cx_Oracle.connect('lb2_teste/lb2@10.200.0.204/oradb')
cur = con.cursor()
#Tempo para que eu possa abrir o trace
time.sleep(60)
cur.execute('truncate table teste')
cur.execute('insert into teste (id) select level from dual connect by level <= 50')
cur.execute('commit')
i = 1
while (i <= 50):
    query = 'select id from teste where id=:id'
    cur.prepare(query)
    cur.execute(None, {'id': i})
    # A cada iteracao para 5 segundos para representar
    # Um tipo de processamento da aplicacao (SQL*Net message from client).
    time.sleep(5)
    i += 1
cur.close()
con.close()
{% endhighlight %}

A aplicação é bem simples, existe uma tabela com apenas uma coluna (ID NUMBER) onde é inserido cinquenta registros de 1 a 50. Um loop é feito pesquisando registro por registro e para cada registro a aplicação 'congela' por cinco segundos. Esses cincos segundos servem somente para representar um tempo em que a aplicação poderia estar processando, por exemplo, fazendo algum tipo de cálculo.

A aplicação vai perder pelo menos 5*49 (245 segundos) segundos 'processando' na máquina local. Vamos executar com um trace para checar se o Oracle encontrou mesmo esse tempo.

# Gerando o trace

Após obter o SID e o serial habilitei o trace incluindo waits e binds.

{% highlight sql %}
execute dbms_monitor.session_trace_enable(session_id=>142,serial_num=>3523,waits=>true,binds=>true);
{% endhighlight %}

Após o termino de execução da aplicação gerei um TRCA (Trace Analyzer) a partir do ".trc".

{% highlight sql %}
SQL> @trcanlzr.sql oradb_ora_26932.trc 
{% endhighlight %}

{% highlight bash %}
TKPROF: Release 11.2.0.3.0 - Development on Qua Mai 14 10:36:40 2014

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.


  adding: trca_e41013.html (deflated 91%)
  adding: trca_e41013.log (deflated 83%)
  adding: trca_e41013_nosort.tkprof (deflated 85%)
  adding: trca_e41013_sort.tkprof (deflated 85%)
  adding: trca_e41013.txt (deflated 88%)
  adding: trcanlzr_error.log (deflated 83%)
test of trca_e41013.zip OK
deleting: trcanlzr_error.log
Archive:  trca_e41013.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
   597007  05-14-2014 10:36   trca_e41013.html
    17202  05-14-2014 10:36   trca_e41013.log
   156026  05-14-2014 10:36   trca_e41013_nosort.tkprof
   156058  05-14-2014 10:36   trca_e41013_sort.tkprof
   340634  05-14-2014 10:36   trca_e41013.txt
---------                     -------
  1266927                     5 files

File trca_e41013.zip has been created

{% endhighlight %}

# Resultado do TRCA

Observem na imagem abaixo que a query levou apenas 0.20 segundos de processamento no banco de dados, o resto do tempo são eventos de wait.

<figure >
	<img src="/images/exemplo02_waits.png" alt="350">
</figure>
Está lá, 245.8 segundos de SQL*Net message from client. O oracle está nos indicando: **Ei, esta vendo este tempo aqui? Eu não tenho nada a ver com isso, não processei nada!**
<figure >
	<img src="/images/exemplo03_waits.png" alt="350">
</figure>

##Conclusão

Muitas vezes esquecemos dos eventos de wait ao realizar um tunning, mas, observem como neste exemplo a análise desses eventos foi crucial para a resolução do problema. 

Em meu problema particular, a partir desta análise, uma equipe de desenvolvedores foi acionada para tentar resolver o problema na aplicação.