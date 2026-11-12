# MariaDB: diagnóstico — onde procurar quando algo está errado

Um guia de **onde olhar** para cada tipo de pergunta: "o que está rodando agora?", "quem está travando?", "cadê o log?". É o mapa que transforma "o banco está lento/travado" em uma causa concreta. Tudo confirmado em MariaDB 12.3.3.

> Veja também: [locks e bloqueios](troubleshooting-locks-e-bloqueios.md), [deadlocks](troubleshooting-deadlocks.md), [monitoramento do Galera](../Galera/galera-monitoramento-observabilidade.md).

## Para quem é este documento

Estagiários e juniores. Guarde isto como "cola de plantão": a pergunta à esquerda, o comando à direita.

## Mapa rápido: pergunta → onde olhar

| Pergunta | Onde olhar |
|---|---|
| O que está rodando agora? | `SHOW FULL PROCESSLIST` / `information_schema.PROCESSLIST` |
| Tem transação aberta/longa? | `information_schema.INNODB_TRX` |
| Quem está travando quem (linha)? | `sys.innodb_lock_waits`, `information_schema.INNODB_LOCK_WAITS` |
| Tem DDL travado por metadata lock? | `performance_schema.metadata_locks` + estado `Waiting for table metadata lock` |
| O que o InnoDB está fazendo por dentro? | `SHOW ENGINE INNODB STATUS` |
| Cadê o log de erro? | `journalctl -u mariadb` |
| Queries lentas? | slow query log |
| É problema de cluster (Galera)? | `SHOW STATUS LIKE 'wsrep_%'` (ver doc de monitoramento) |

> **Nesta instalação não existe `/var/log/mysql/error.log`** — o log do MariaDB fica no `journalctl -u mariadb`.

---

## 1. O que está rodando — `PROCESSLIST`

```sql
SHOW FULL PROCESSLIST;
-- ou, como tabela (dá para filtrar e ordenar):
SELECT id, user, host, db, command, time, state, LEFT(info,60) AS query
FROM information_schema.PROCESSLIST
WHERE command <> 'Sleep'
ORDER BY time DESC;
```

O que observar:

- **`command`**: `Query` (executando), `Sleep` (conexão ociosa — cuidado com muitas `Sleep` com `time` alto: conexões vazando).
- **`time`**: segundos no estado atual. Query com `time` alto = candidata a investigar.
- **`state`**: o que a query está fazendo. Estados úteis em troubleshooting:
  - `Updating` / `Sending data` — executando.
  - `Waiting for table metadata lock` — travada por um DDL/transação (ver [locks](troubleshooting-locks-e-bloqueios.md)).
  - `User sleep` — dentro de um `SLEEP()`.
  - `Waiting for ...` — quase sempre esperando algum lock/recurso.

> Em Galera você verá conexões `system user` com estados `wsrep applier idle` / `wsrep aborter idle` — são **internas do cluster**, normais, não mexa nelas.

## 2. Transações abertas e longas — `INNODB_TRX`

A causa nº 1 de bloqueio é **transação aberta esquecida**. Ache as mais antigas:

```sql
SELECT trx_id, trx_state, trx_mysql_thread_id AS conn,
       trx_started, TIMESTAMPDIFF(SECOND, trx_started, NOW()) AS idade_seg,
       trx_rows_locked, trx_rows_modified, LEFT(trx_query,50) AS query
FROM information_schema.INNODB_TRX
ORDER BY trx_started;
```

- `trx_state = LOCK WAIT` → está esperando um lock (vá para o passo 3).
- `idade_seg` alto com `trx_query = NULL` → transação **aberta e parada** (aplicação não deu `COMMIT`). É a vilã clássica.

## 3. Quem trava quem (locks de linha)

A forma direta, com o comando de KILL pronto:

```sql
SELECT waiting_pid, waiting_query, blocking_pid, blocking_query,
       sql_kill_blocking_connection
FROM sys.innodb_lock_waits\G
```

As views cruas por trás disso (úteis para scripts):

```sql
SELECT requesting_trx_id, blocking_trx_id
FROM information_schema.INNODB_LOCK_WAITS;

SELECT * FROM information_schema.INNODB_LOCKS;
```

> **MariaDB x MySQL 8:** aqui as views `information_schema.INNODB_LOCKS` e `INNODB_LOCK_WAITS` **existem**. No MySQL 8 elas foram removidas (lá se usa `performance_schema.data_locks`, que **não existe** no MariaDB). Não copie receita de MySQL 8 sem checar.

## 4. Metadata locks (DDL travado)

```sql
-- O instrumento vem DESLIGADO por padrão; ligue uma vez:
UPDATE performance_schema.setup_instruments
SET ENABLED='YES', TIMED='YES'
WHERE NAME='wait/lock/metadata/sql/mdl';

-- Depois, consulte:
SELECT ml.object_schema, ml.object_name, ml.lock_type, ml.lock_status,
       t.processlist_id AS conn
FROM performance_schema.metadata_locks ml
JOIN performance_schema.threads t ON t.thread_id = ml.owner_thread_id
WHERE ml.object_schema NOT IN ('mysql','performance_schema','information_schema');
```

`lock_status = GRANTED` é quem segura; `PENDING` é quem espera. Sinal no `PROCESSLIST`: estado `Waiting for table metadata lock`.

> **Em Galera**, lembre: um DDL não espera — ele aborta a transação local (BF-abort). Ver o Cenário 2 em [locks e bloqueios](troubleshooting-locks-e-bloqueios.md).

## 5. Por dentro do InnoDB — `SHOW ENGINE INNODB STATUS`

Um despejo do estado do motor. Seções mais úteis:

```sql
SHOW ENGINE INNODB STATUS\G
```

- **`LATEST DETECTED DEADLOCK`** — o último deadlock, com as duas transações (ver [deadlocks](troubleshooting-deadlocks.md)).
- **`TRANSACTIONS`** — lista de transações ativas e locks.
- **`BUFFER POOL AND MEMORY`**, **`ROW OPERATIONS`** — pressão de memória/IO.

## 6. Logs

```bash
# Log de erro (crashes, warnings, deadlocks se innodb_print_all_deadlocks=ON)
sudo journalctl -u mariadb -n 100 --no-pager
sudo journalctl -u mariadb --since '15 min ago' --no-pager -o cat

# Seguir em tempo real
sudo journalctl -u mariadb -f
```

### Slow query log (queries lentas)

```sql
-- Ver config
SHOW VARIABLES LIKE 'slow_query_log%';
SHOW VARIABLES LIKE 'long_query_time';

-- Ligar em runtime (queries acima de 1s)
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;
```

Depois analise com `mariadb-dumpslow` (agrega as queries mais lentas do arquivo).

## 7. Matar o que está travado

```sql
KILL <id>;          -- encerra a CONEXÃO (reverte a transação)
KILL QUERY <id>;    -- mata só a QUERY atual (mantém a conexão/transação)
```

Pegue o `<id>` na coluna `id` do `PROCESSLIST` ou `blocking_pid` do `sys.innodb_lock_waits`. **Sempre confirme que é o bloqueador**, não a vítima.

## 8. Variáveis e contadores úteis

```sql
-- Timeouts de lock
SHOW VARIABLES WHERE Variable_name IN ('innodb_lock_wait_timeout','lock_wait_timeout');
--   innodb_lock_wait_timeout = locks de LINHA (padrão 50s)
--   lock_wait_timeout        = metadata locks / DDL (padrão 86400s!)

-- Detecção de deadlock e log
SHOW VARIABLES WHERE Variable_name IN ('innodb_deadlock_detect','innodb_print_all_deadlocks');

-- Conexões
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Threads_running';    -- running alto = contenção

-- Galera (cluster)
SHOW STATUS WHERE Variable_name IN ('wsrep_local_bf_aborts','wsrep_cluster_status','wsrep_ready');
```

> Atenção ao `lock_wait_timeout` (metadata) ter padrão **enorme** (1 dia): um `ALTER` travado por metadata lock num servidor standalone pode "pendurar" por muito tempo. Em sessão de manutenção, reduza-o (`SET SESSION lock_wait_timeout = 30;`) antes de rodar DDL.

## O schema `sys` (atalhos prontos)

O schema `sys` existe nesta instalação e tem views que resumem tudo:

```sql
SELECT * FROM sys.innodb_lock_waits\G       -- bloqueios de linha (com KILL pronto)
SELECT * FROM sys.processlist\G             -- processlist enriquecido
SELECT * FROM sys.schema_table_lock_waits\G -- esperas por lock de tabela/metadata
```

## Roteiro de plantão: "o banco travou"

1. **`PROCESSLIST`** — há queries com `time` alto ou estado `Waiting for...`?
2. **`INNODB_TRX`** — há transação `LOCK WAIT` ou aberta e parada há muito tempo?
3. **`sys.innodb_lock_waits`** — quem é o `blocking_pid`?
4. **Decisão:** esperar, avisar o dono, ou `KILL <blocking_pid>`.
5. Se foi **deadlock** (erro `1213`): `SHOW ENGINE INNODB STATUS` → ver o padrão e corrigir a aplicação.
6. Se é **cluster**: `SHOW STATUS LIKE 'wsrep_%'` (ver [monitoramento do Galera](../Galera/galera-monitoramento-observabilidade.md)).

## Veja também

- [Locks e bloqueios](troubleshooting-locks-e-bloqueios.md)
- [Deadlocks](troubleshooting-deadlocks.md)
- [Monitoramento do Galera](../Galera/galera-monitoramento-observabilidade.md)
