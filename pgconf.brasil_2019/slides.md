---
title      : Escalando PostgreSQL 11
author     : Daniel Naves de Carvalho
theme: league
revealOptions:
    transition: 'fade'
---

# Escalando PostgreSQL 11
#### Como escalar PostgreSQL 11 utilizando partitioning e fdw

<small> [Daniel Naves de Carvalho](http://www.wearebrane.com/) / Github: [@danielnc](http://github.com/danielnc) </small>

---
# O que queriamos resolver

Notes:
* Database crescendo exponencialmente a cada dia
* Usuarios em quase a totalidade dos estados dos EUA
* Tabelas e indexes não cabem mais em memória de uma única máquina
* Mais e mais clientes(horizontal scaling) conectam nos servidores a cada dia
* Operações por segundo crescendo(insert/updates/deletes)
* Minoria dos usuários responsávels pela maioria dos dados e operações/segundo

---

## Arquitetura simplificada sistema
![](imgs/database.png)

---

## O que foi proposto

* Database precisa escalar horizontalmente
* Usuários maiores não afetam os demais
* Otimizar dados e indexes
* Acesso sem delay a usuários
* Analise de dados cross-region

Notes:
* Usuários maiores ficam isolados na propria instancia diminuindo o resto de afetar usuarios menores
* Dados e indexes devem caber em memória
* Usuarios tem a aplicação e o banco de dados o mais próximo possivel
* Queremos continuar cruzando dados de usuários cross-region

---

## PostgreSQL 11 - Partitioning

* O que é?
* Beneficios

Notes:
O que é:
* Quebrar o que é logicamente uma tabela grande em pedaços fisicos menores

Beneficios:
* Performance de queries pode ser melhorada drasticamente em certos cenários, principalmente quando table rows que sao acessadas frequentemente ficam em um grupo pequeno de partições.
* Queries ou updates que acessam uma grande parte de uma unica partição podem utilizar scan sequencial em vez de utilizar indexes em tabelas maiores e fazer random access para encontrar os dados
* Bulk load/delete pode ser feito adicionando ou removendo partições. Isto evida o overhead do VACUUM que é causado por exemplo, por um bulk delete
* Dados pouco utilizado pode ser migrado para um storage mais lento/barato

---

## Tipos de Partitioning

* Range Partitioning
* List Partitioning
* Hash Partitioning

Notes:
Range -> ranges sem overlap, por exemplo date ranges
List: lista de valores que aparecem em cada partição
Hash: Cada partição fica com os valores que o valor do hash é o resto da divisão do modulo.

---

## SQL - Tabela

```SQL
=# CREATE TABLE orders (
(#     id integer NOT NULL,
...
(#     client_id integer
(# ) PARTITION BY HASH (client_id);
```

---

## SQL - Criando as partições

```SQL
=# CREATE TABLE orders_0 PARTITION OF orders FOR VALUES WITH (MODULUS 10, REMAINDER 0);
=# CREATE TABLE orders_1 PARTITION OF orders FOR VALUES WITH (MODULUS 10, REMAINDER 1);
...
=# CREATE TABLE orders_9 PARTITION OF orders FOR VALUES WITH (MODULUS 10, REMAINDER 9);
```

---

## SQL - Analize

```
# explain analyze select count(1) from orders;
       QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=709533.57..709533.58 rows=1 width=8) (actual time=2637.131..2637.131 rows=1 loops=1)
   ->  Gather  (cost=709533.35..709533.56 rows=2 width=8) (actual time=2636.897..2716.635 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=708533.35..708533.36 rows=1 width=8) (actual time=2632.932..2632.932 rows=1 loops=3)
               ->  Parallel Append  (cost=0.00..684754.30 rows=9511620 width=0) (actual time=0.025..2181.186 rows=7609378 loops=3)
                     ->  Parallel Seq Scan on orders_3  (cost=0.00..136556.25 rows=2019325 width=0) (actual time=0.014..1174.520 rows=4846509 loops=1)
                     ->  Parallel Seq Scan on orders_1  (cost=0.00..80967.66 rows=1214766 width=0) (actual time=0.020..705.673 rows=2915745 loops=1)
                     ->  Parallel Seq Scan on orders_4  (cost=0.00..65960.76 rows=975976 width=0) (actual time=0.017..535.281 rows=2342430 loops=1)
                     ->  Parallel Seq Scan on orders_5  (cost=0.00..62335.75 rows=940175 width=0) (actual time=0.012..167.984 rows=752182 loops=3)
                     ->  Parallel Seq Scan on orders_6  (cost=0.00..61256.48 rows=913148 width=0) (actual time=0.014..240.916 rows=1095756 loops=2)
                     ->  Parallel Seq Scan on orders_8  (cost=0.00..59717.07 rows=909907 width=0) (actual time=0.009..363.421 rows=2184165 loops=1)
                     ->  Parallel Seq Scan on orders_9  (cost=0.00..57672.90 rows=860390 width=0) (actual time=0.008..335.504 rows=2064304 loops=1)
                     ->  Parallel Seq Scan on orders_7  (cost=0.00..47275.52 rows=701252 width=0) (actual time=0.009..282.327 rows=1683089 loops=1)
                     ->  Parallel Seq Scan on orders_2  (cost=0.00..44870.31 rows=669531 width=0) (actual time=0.009..278.900 rows=1606672 loops=1)
                     ->  Parallel Seq Scan on orders_0  (cost=0.00..20583.50 rows=307150 width=0) (actual time=0.036..125.200 rows=737161 loops=1)
 Planning Time: 0.228 ms
 Execution Time: 2716.726 ms
(18 rows)
```

---

## SQL - Partições

```SQL
# SELECT tableoid::regclass AS partition_name, COUNT(*) FROM orders GROUP BY 1 ORDER BY 1;
     partition_name     |  count
------------------------+---------
 orders_0 |  737161
 orders_1 | 2915745
 orders_2 | 1606672
 orders_3 | 4846509
 orders_4 | 2342430
 orders_5 | 2256547
 orders_6 | 2191512
 orders_7 | 1683089
 orders_8 | 2184165
 orders_9 | 2064304
(10 rows)
```

Notes: partição 3 é a maior
---

## SQL - Rebalancing partitions - Parte 1

```SQL
=# CREATE TABLE orders_3_0 (LIKE orders);
=# CREATE TABLE orders_3_1 (LIKE orders);
=# CREATE TABLE orders_3_2 (LIKE orders);
```
---

## SQL - Rebalancing partitions - Parte 2

```SQL
=# WITH moved AS (
(# DELETE FROM orders_3
(# WHERE satisfies_hash_partition('orders'::regclass, 30, 3,
(# client_id)
(# RETURNING *)
-# INSERT INTO orders_3_0 SELECT * FROM moved;
=#
=# WITH moved AS (
(# DELETE FROM orders_3
(# WHERE satisfies_hash_partition('orders'::regclass, 30, 13,
(# client_id)
(# RETURNING *)
-# INSERT INTO orders_3_1 SELECT * FROM moved;
=# WITH moved AS (
(# DELETE FROM orders_3
(# WHERE satisfies_hash_partition('orders'::regclass, 30, 23,
(# client_id)
(# RETURNING *)
-# INSERT INTO orders_3_2 SELECT * FROM moved;
```
---
## SQL - Rebalancing partitions - Parte 3
```SQL
=# ALTER TABLE orders DETACH PARTITION orders_3;
=# ALTER TABLE orders ATTACH PARTITION orders_3_0 FOR VALUES WITH (MODULUS 30, REMAINDER 3);
=# ALTER TABLE orders ATTACH PARTITION orders_3_1 FOR VALUES WITH (MODULUS 30, REMAINDER 13);
=# ALTER TABLE orders ATTACH PARTITION orders_3_2 FOR VALUES WITH (MODULUS 30, REMAINDER 23);
```

---
## SQL - Rebalancing partitions - Parte 4

```SQL
=# SELECT tableoid::regclass AS partition_name, COUNT(*) FROM orders GROUP BY 1 ORDER BY 1;
      partition_name      |  count
--------------------------+---------
 orders_0   |  737161
 orders_1   | 2915745
 orders_2   | 1606672
 orders_4   | 2342430
 orders_5   | 2256547
 orders_6   | 2191512
 orders_7   | 1683089
 orders_8   | 2184165
 orders_9   | 2064304
 orders_3_0 | 1535006
 orders_3_1 | 1146945
 orders_3_2 | 2164558
 ```
---

## SQL - Resultados


### Tabela antiga
```SQL
# select count(1) from old_orders where client_id = 111;
  count
---------
 1335788
(1 row)
Time: 1573.653 ms
```


### Tabela nova
```SQL
# select count(1) from orders where client_id = 111;
  count
---------
 1335788
(1 row)
Time: 63.048 ms
```
---
## Aproximadamente 24x mais rápido

---

## Foreign Data Wrapers

* NoSQL adapters, File Adapters, LDAP Adapters, Operating System adapters, etc
* https://wiki.postgresql.org/wiki/Foreign_data_wrappers

Notes:
* FDWs possibilitam acessar e manipular dados externos ao PostgreSQL de dentro banco de dados.

---

## Utilizando FDWs com tabelas particionadas

```SQL
=# CREATE EXTENSION postgres_fdw;

=# CREATE SERVER server2 FOREIGN DATA WRAPPER postgres_fdw OPTIONS (host 'server2', dbname 'other_db');

=# CREATE USER MAPPING FOR danielnc SERVER server2 OPTIONS (user 'server2_danielnc');

=# CREATE FOREIGN TABLE orders_0 PARTITION OF orders  FOR VALUES WITH (MODULUS 10, REMAINDER 0) SERVER server2;
```

---

## Parametros que requerem atenção

* enable_partitionwise_aggregate: default false
* fetch_size: default 100

Notes:
* enable_partitionwise_aggregate -> Se a partitiom key é a mesma do group by key, cada partição cria um grupo discreto de grupos em vez do scan acontecer em todas as particoes ao mesmo tempo. Acontece um parallel aggregate para cada particao e no passo final, os resultados sao concatenados
* fetch_size -> Determina o numero de linhas que o fdw ira enviar por operacao. pode ser setado por tabela ou server. tabela sobreescreve o server

---
## Nosso presente e nosso futuro

Notes:
Com a ajuda de partitioning e FDWs, criamos um servidor central que cria as tabelas e servidores secundários em algumas diferentes regiões, que sao responsaveis pelas transações daquela região.
Podemos agregar os resultados de maneira transparente no servidor central devido FDW e partitioning, e para os clientes, os servidores secundarios sao mais rápidos e performam melhor

---
![](imgs/database_future.png)
---
## Perguntas?

<small> [Daniel Naves de Carvalho](http://www.wearebrane.com/) / Github: [@danielnc](http://github.com/danielnc) </small>

<small> [daniel@wearebrane.com](mailto:daniel@wearebrane.com) </small>

### Estamos contratando!
