# MariaDB: locks e bloqueios — como identificar e resolver

Quando uma query "trava" sem erro e fica esperando, quase sempre é **bloqueio por lock**: outra transação está segurando o que a sua precisa. Este documento mostra, com cenários reais executados em laboratório, como enxergar quem está bloqueando quem e como resolver.

> Procedimentos **executados e validados** em um nó MariaDB 12.3.3 (cluster Galera, db-01). As saídas são reais. Veja também: [deadlocks](troubleshooting-deadlocks.md) e [onde procurar / diagnóstico](troubleshooting-diagnostico-onde-procurar.md).

## Para quem é este documento

Estagiários e juniores. A meta: diante de "minha query está pendurada", você saber em 3 comandos **quem é o culpado** e decidir o que fazer.

## Conceitos rápidos

O InnoDB usa vários tipos de lock. Os que mais aparecem em troubleshooting:

- **Lock de linha (row lock):** protege linhas específicas durante `UPDATE`/`DELETE`/`INSERT` e `SELECT ... FOR UPDATE`. Dois modos principais: **X** (exclusivo, escrita) e **S** (compartilhado). Uma transação segura o lock até dar `COMMIT`/`ROLLBACK`.
- **Metadata lock (MDL):** protege a *estrutura* de uma tabela. Qualquer transação que toca a tabela segura um MDL sobre ela até terminar; um `ALTER`/`DROP` precisa de MDL **exclusivo**.
- **Lock de linha ≠ deadlock.** Bloqueio (este documento) é uma transação **esperando** outra. Deadlock ([outro documento](troubleshooting-deadlocks.md)) é um **abraço mortal** que o InnoDB detecta e desfaz sozinho, abortando uma delas.

## Objetos de teste

Todos os cenários usam este schema (criado no db-01, replicado pelo Galera):

```sql
CREATE DATABASE lab_tshoot;
CREATE TABLE lab_tshoot.contas (
  id INT PRIMARY KEY,
  titular VARCHAR(40) NOT NULL,
  saldo DECIMAL(12,2) NOT NULL,
  atualizado_em TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB;
INSERT INTO lab_tshoot.contas (id, titular, saldo) VALUES
 (1,'Ana',1000),(2,'Bruno',2000),(3,'Carla',3000),(4,'Diego',4000),(5,'Eva',5000);
```

---

## Cenário 1: bloqueio de linha (o caso mais comum)

**Montagem:** a sessão **A** abre uma transação, faz `UPDATE` na linha `id=1` e **não commita** (fica parada). A sessão **B** tenta alterar a mesma linha e **fica esperando**.

Sessão A (segura o lock):
```sql
START TRANSACTION;
UPDATE lab_tshoot.contas SET saldo = saldo - 100 WHERE id = 1;
-- (não commita; fica aqui)
```

Sessão B (fica bloqueada):
```sql
START TRANSACTION;
UPDATE lab_tshoot.contas SET saldo = saldo + 100 WHERE id = 1;   -- PENDURADA
```

### Passo 1: ver que há uma transação esperando — `INNODB_TRX`

```sql
SELECT trx_id, trx_state, trx_mysql_thread_id AS conn, trx_rows_locked,
       trx_wait_started, LEFT(trx_query,45) AS query
FROM information_schema.INNODB_TRX\G
```

Saída real:

```
*** 1. row *** trx_id: 259  trx_state: LOCK WAIT  conn: 319
   trx_wait_started: 2026-11-11 21:50:18   query: UPDATE lab_tshoot.contas SET saldo = saldo +
*** 2. row *** trx_id: 258  trx_state: RUNNING    conn: 312
   trx_wait_started: NULL                  query: SELECT SLEEP(30)
```

Já se vê: a transação **259 está em `LOCK WAIT`** (esperando), a **258 está `RUNNING`** (a dona do lock).

### Passo 2: quem bloqueia quem — `INNODB_LOCK_WAITS`

```sql
SELECT requesting_trx_id AS esperando, blocking_trx_id AS bloqueando
FROM information_schema.INNODB_LOCK_WAITS;
```

```
+-----------+------------+
| esperando | bloqueando |
+-----------+------------+
|       259 |        258 |
+-----------+------------+
```

> **Importante (diferença para o MySQL 8):** o MariaDB **mantém** `information_schema.INNODB_LOCKS` e `INNODB_LOCK_WAITS`. O MySQL 8 removeu essas views e usa `performance_schema.data_locks` — que **não existe** no MariaDB. Aqui, use as views `INNODB_*`.

### Passo 3: a forma mais fácil — `sys.innodb_lock_waits`

A view do schema `sys` já entrega tudo mastigado, **incluindo o comando de KILL pronto**:

```sql
SELECT waiting_pid, waiting_query, blocking_pid, blocking_query,
       sql_kill_blocking_connection
FROM sys.innodb_lock_waits\G
```

Saída real:

```
        waiting_pid: 319
      waiting_query: UPDATE lab_tshoot.contas SET saldo = saldo + 100 WHERE id = 1
       blocking_pid: 312
     blocking_query: SELECT SLEEP(30)
sql_kill_blocking_connection: KILL 312
```

O `SHOW [FULL] PROCESSLIST` confirma o estado:

```
| 312 | Query | 7 | User sleep | SELECT SLEEP(30)   |   <- dono do lock
| 319 | Query | 5 | Updating   | UPDATE ... id = 1  |   <- esperando (state 'Updating')
```

### Passo 4: resolver

Três saídas, da mais educada para a mais drástica:

1. **Esperar** o dono commitar/encerrar (se for uma transação legítima e curta).
2. **Deixar estourar o timeout**: por padrão `innodb_lock_wait_timeout = 50` segundos; a query bloqueada falha com `ERROR 1205: Lock wait timeout exceeded`.
3. **Matar o bloqueador** (quando é uma transação esquecida/zumbi):
   ```sql
   KILL 312;
   ```

No lab, após `KILL 312`, a sessão B destravou na hora e commitou — o `-100` do bloqueador foi revertido (rollback ao matar) e o `+100` da vítima persistiu:

```
VITIMA CONCLUIU O UPDATE
| id | titular | saldo   |
|  1 | Ana     | 1100.00 |
```

> **`KILL` vs `KILL QUERY`:** `KILL <id>` encerra a **conexão** inteira (e reverte a transação). `KILL QUERY <id>` mata só a query atual, mantendo a conexão e a transação — útil quando você quer interromper um comando sem derrubar a sessão.

---

## Cenário 2: metadata lock (DDL travado por transação aberta)

Uma transação aberta que só **leu** uma tabela já segura um MDL sobre ela. Um `ALTER`/`DROP` precisa de MDL exclusivo e, em um servidor **standalone**, fica no estado `Waiting for table metadata lock`.

### Pré-requisito surpresa: o instrumento de MDL vem desligado

No MariaDB, a view `performance_schema.metadata_locks` só popula se o instrumento estiver ligado — e **o padrão é desligado**:

```sql
SELECT NAME, ENABLED, TIMED FROM performance_schema.setup_instruments
WHERE NAME = 'wait/lock/metadata/sql/mdl';
--  -> ENABLED = NO   (padrão)
```

Ligue (vale até o restart):

```sql
UPDATE performance_schema.setup_instruments
SET ENABLED='YES', TIMED='YES'
WHERE NAME='wait/lock/metadata/sql/mdl';
```

Agora, com uma transação segurando a tabela, a view mostra o lock:

```sql
SELECT ml.object_name, ml.lock_type, ml.lock_status, t.processlist_id AS conn
FROM performance_schema.metadata_locks ml
JOIN performance_schema.threads t ON t.thread_id = ml.owner_thread_id
WHERE ml.object_schema = 'lab_tshoot';
```

Saída real (transação aberta que leu `contas`):

```
| object_name | lock_type    | lock_status |
| contas      | SHARED_WRITE | GRANTED     |
```

### ⚠️ Em Galera, o DDL NÃO espera o metadata lock — ele aborta a transação

Este é um comportamento **específico de cluster** que pega muita gente. Em um servidor standalone, o `ALTER` ficaria em `Waiting for table metadata lock`. No Galera, o DDL roda por **TOI (Total Order Isolation)** e tem prioridade: ele **aborta à força (BF-abort)** a transação local que estiver no caminho.

Evidência real do lab — com uma transação segurando a linha e a tabela, rodamos o `ALTER`:

```
# ALTER cronometrado:
real    0m0.080s           <- NÃO esperou; concluiu na hora

# a transação que segurava recebeu:
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction

# contador de BF-aborts do Galera subiu:
wsrep_local_bf_aborts: 3 -> 4

# e a alteração dela foi revertida (saldo voltou ao valor original)
```

> **O que aprender:** num cluster Galera, um DDL (mesmo via ferramentas) pode **derrubar transações da aplicação** com erro `1213`/BF-abort, em vez de simplesmente esperar. Por isso DDL em produção com Galera é feito em janela controlada. O contador `wsrep_local_bf_aborts` subindo é o sinal de que isso está acontecendo. Veja [monitoramento do Galera](../Galera/galera-monitoramento-observabilidade.md).

---

## Resumo: achar e matar um bloqueador em 2 comandos

```sql
-- 1. Quem espera x quem bloqueia (com o KILL pronto)
SELECT waiting_pid, blocking_pid, blocking_query, sql_kill_blocking_connection
FROM sys.innodb_lock_waits;

-- 2. Matar o bloqueador (use o id da coluna blocking_pid)
KILL <blocking_pid>;
```

## O que NÃO fazer

- **Não saia matando conexões.** Confirme no `sys.innodb_lock_waits`/`INNODB_TRX` quem é o bloqueador antes. Matar a vítima não resolve nada.
- **Não aumente o `innodb_lock_wait_timeout` para "resolver".** Timeout alto esconde o problema; a causa costuma ser transação aberta sem commit na aplicação (autocommit desligado, conexão vazando).
- **Não confunda bloqueio com deadlock.** Bloqueio fica esperando (até timeout); deadlock o InnoDB já resolve sozinho com erro `1213`. Ver [deadlocks](troubleshooting-deadlocks.md).
- **Em Galera, não rode DDL casualmente em produção** — ele pode abortar transações da aplicação.

## Veja também

- [Deadlocks](troubleshooting-deadlocks.md)
- [Onde procurar / diagnóstico geral](troubleshooting-diagnostico-onde-procurar.md)
- [Monitoramento do Galera](../Galera/galera-monitoramento-observabilidade.md)
