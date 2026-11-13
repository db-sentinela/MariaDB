# 02 — Thread Pool: Escalabilidade e Concorrência Nativa

Em arquiteturas tradicionais de banco de dados, cada conexão aberta pelo cliente aloca uma thread dedicada do sistema operacional (`one-thread-per-connection`). Sob alta carga (centenas ou milhares de conexões simultâneas), esse modelo entra em colapso devido à sobrecarga de chaveamento de contexto (*context switching*) na CPU e ao esgotamento de memória. No MariaDB, o **Thread Pool** é um recurso nativo avançado que desacopla conexões de clientes das threads de execução da CPU.

---

## O que é

O **Thread Pool** substitui o despachante tradicional por uma arquitetura assíncrona orientada a grupos de threads de trabalho:
- **Thread Groups:** O pool é dividido em grupos independentes (controlados por `thread_pool_size`, geralmente configurado para o número de vCPUs da máquina). Cada grupo atua como um subsistema fechado.
- **Listener Thread:** Dentro de cada grupo, uma thread atua como ouvinte usando chamadas nativas de alta performance do kernel Linux (`epoll`), aguardando a chegada de instruções SQL.
- **Worker Threads:** Ao receber uma consulta, o ouvinte acorda ou cria uma thread de trabalho para processar a requisição. Ao terminar a query, a conexão retorna à fila e a worker thread fica livre para atender outro cliente.
- **Oversubscription:** O parâmetro `thread_pool_oversubscribe` define quantas threads de trabalho ativas podem rodar simultaneamente em cada grupo antes de segurar novas requisições na fila.

Ao contrário do MySQL Community (onde o Thread Pool é restrito à versão paga Enterprise), no MariaDB o Thread Pool é aberto, embutido no core e altamente ajustável.

---

## Por que importa

1. **Proteção contra Quedas por Avalanches de Conexões:** Quando uma aplicação web sofre um pico (ex: 2.000 clientes simultâneos), o modelo tradicional aloca 2.000 threads. A CPU passa a gastar 80% do seu tempo trocando de contexto entre threads em vez de processar queries, levando o banco a timeouts generalizados. Com o Thread Pool, a vazão de queries se mantém linear e previsível.
2. **Economia Brutal de Memória RAM:** No modelo tradicional, cada thread reserva no mínimo `thread_stack` (292 KB no MariaDB) além de buffers privados (`sort_buffer_size`, `read_buffer_size`, `join_buffer_size`). Com 2.000 conexões, apenas a infraestrutura de threads consome vários gigabytes. O Thread Pool atende as mesmas 2.000 conexões com apenas algumas dezenas de threads de trabalho em execução.
3. **Prevenção de Starvation com Stall Limit:** Se uma query longa rodar em uma thread de trabalho, o mecanismo `thread_pool_stall_limit` detecta a estagnação e despacha uma nova thread para não bloquear consultas curtas e rápidas.

---

## Como reproduzir

Execute os passos a seguir no laboratório para configurar, ativar e testar o Thread Pool no cluster:

### 1. Verificar o comportamento somente leitura em runtime
Tentar alterar o despachante dinamicamente via terminal:
```sql
SET GLOBAL thread_handling = 'pool-of-threads';
```
*(Observe o erro gerado, provando que a ativação requer configuração estática).*

### 2. Criar o arquivo de configuração no MariaDB
No nó `mariadb-db-01` (e replicado no `mariadb-db-02`), crie o arquivo `/etc/mysql/mariadb.conf.d/80-tuning.cnf`:

```ini
[mariadb]
# Habilita o despachante assincrono de Thread Pool
thread_handling = pool-of-threads

# Define o numero de grupos de threads (igual ao numero de vCPUs)
thread_pool_size = 4

# Tempo maximo (ms) que uma query pode rodar antes de criar thread adicional
thread_pool_stall_limit = 500

# Limite de threads ativas por grupo
thread_pool_oversubscribe = 3

# Tempo de ociosidade (segundos) antes de encerrar worker threads sobressalentes
thread_pool_idle_timeout = 60
```

### 3. Aplicar via Rolling Restart sem indisponibilidade
Em um cluster Galera com 3 nós (2 dados + 1 árbitro), aplique o reinício em um nó por vez:
```bash
sudo systemctl restart mariadb
sudo mariadb -e "SHOW STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_local_state_comment');"
```
*(Aguarde o nó restabelecer `Synced` e repita o processo no segundo nó).*

### 4. Simular carga de alta concorrência e monitorar grupos
Dispare uma rajada de conexões simultâneas em background:
```bash
for i in {1..20}; do
    mariadb -e "SELECT SLEEP(1);" > /dev/null 2>&1 &
done
```

Durante a execução da rajada, inspecione a distribuição de conexões pelas tabelas de sistema:
```sql
SELECT 
    GROUP_ID, CONNECTIONS, THREADS, ACTIVE_THREADS, 
    STANDBY_THREADS, QUEUE_LENGTH, IS_STALLED 
FROM information_schema.THREAD_POOL_GROUPS;

SHOW STATUS LIKE 'Threadpool%';
```

---

## O que você vê (saída real)

### Tentativa de ativação dinâmica em runtime:
```text
mariadb -e "SET GLOBAL thread_handling='pool-of-threads';"
ERROR 1238 (HY000) at line 1: Variable 'thread_handling' is a read only variable
```

### Parâmetros consolidados após restart do serviço:
```sql
mariadb -e "SELECT @@thread_handling, @@thread_pool_size, @@thread_pool_stall_limit;"
```
Saída real:
```text
+-------------------+--------------------+--------------------------+
| @@thread_handling | @@thread_pool_size | @@thread_pool_stall_limit|
+-------------------+--------------------+--------------------------+
| pool-of-threads   | 4                  | 500                      |
+-------------------+--------------------+--------------------------+
```

### Distribuição equilibrada durante o pico de 20 conexões simultâneas:
```sql
SELECT GROUP_ID, CONNECTIONS, THREADS, ACTIVE_THREADS, STANDBY_THREADS, QUEUE_LENGTH, IS_STALLED 
FROM information_schema.THREAD_POOL_GROUPS;
```
Saída real:
```text
+----------+-------------+---------+----------------+-----------------+--------------+------------+
| GROUP_ID | CONNECTIONS | THREADS | ACTIVE_THREADS | STANDBY_THREADS | QUEUE_LENGTH | IS_STALLED |
+----------+-------------+---------+----------------+-----------------+--------------+------------+
|        0 |           5 |       6 |              0 |               0 |            0 |          0 |
|        1 |           6 |       7 |              1 |               1 |            0 |          0 |
|        2 |           5 |       6 |              0 |               0 |            0 |          0 |
|        3 |           5 |       6 |              0 |               0 |            0 |          0 |
+----------+-------------+---------+----------------+-----------------+--------------+------------+
```

### Contadores de threads do pool:
```sql
SHOW STATUS LIKE 'Threadpool%';
```
Saída real:
```text
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| Threadpool_idle_threads  | 25    |
| Threadpool_threads       | 26    |
+--------------------------+-------+
```

---

## Lições

- **Dimensionamento Ideal de `thread_pool_size`:** Deve ser rigorosamente igual ao número de núcleos lógicos de processamento da máquina física ou VM (`nproc`). Em nosso ambiente com 4 vCPUs, `thread_pool_size = 4`. Aumentar esse valor além da contagem de CPUs reintroduz a contenção por agendamento de threads no kernel.
- **O Papel do `thread_pool_oversubscribe`:** O valor padrão (3) permite que até $3 + 1 = 4$ threads ativas trabalhem em um grupo se tarefas entrarem em espera de I/O. Isso evita que a CPU fique ociosa caso uma thread esteja aguardando leitura de disco.
- **Tabelas Oficiais do Information Schema:** As métricas do pool no MariaDB 12 são lidas em `information_schema.THREAD_POOL_GROUPS`, `THREAD_POOL_QUEUES`, `THREAD_POOL_STATS` e `THREAD_POOL_WAITS`.

---

## Pegadinhas

- **`thread_handling` Não É Dinâmico:** Nunca tente executar scripts de automação que façam `SET GLOBAL thread_handling`. A variável é puramente estática e qualquer tentativa abortará com `ERROR 1238 (HY000)`. Mudanças exigem alteração no arquivo `.cnf` e reinicialização controlada.
- **A Tabela Inexistente `THREAD_POOL_STATUS`:** Documentações antigas ou manuais de plugins legados de terceiros citam uma tabela `information_schema.THREAD_POOL_STATUS`. No MariaDB moderno, essa tabela **não existe** (disparando `ERROR 1109 (42S02)`). Utilize sempre `THREAD_POOL_GROUPS`.
- **Ajuste Excessivamente Baixo do `thread_pool_stall_limit`:** Se você reduzir o `stall_limit` para valores extremos (ex: 10 ms), qualquer query analítica ligeiramente mais demorada disparará a criação de dezenas de novas threads, anulando o propósito do pool e gerando picos artificiais de criação de threads. Mantenha entre 300 ms e 600 ms.
