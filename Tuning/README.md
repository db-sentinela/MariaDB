# DB Sentinel | MariaDB — Tuning & Otimização de Performance

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração de bancos de dados em ambientes de missão crítica.

Esta subpasta cobre o **Tuning e Otimização Avançada do MariaDB 12** em topologia de alta disponibilidade (Cluster Galera com 2 nós de dados, árbitro garbd, HAProxy e Keepalived), abrangendo tanto a **camada de engine/instância** (Buffer Pool, Thread Pool, Redo Log, Galera, Plugins, Tabelas Temporárias) quanto a **camada de consultas e otimizador de custos** (EXPLAIN/ANALYZE JSON, Índices Compostos, ICP, Covering Index, Filesort, Paginação Keyset, Joins e MRR). Todos os procedimentos, fórmulas matemáticas de dimensionamento, parâmetros de engines, ajustes dinâmicos e planos de execução foram **executados, mensurados e validados no laboratório real** antes da consolidação desta documentação.

---

## Procedimentos

### Parte 1: Tuning de Instância, Memória e Alta Disponibilidade
| Procedimento | Descrição | Status |
|---|---|---|
| [01 — InnoDB Buffer Pool & Memória](01-tuning-innodb-buffer-pool-memoria.md) | Dimensionamento, Hit Ratio real, Dirty Pages flushing, redimensionamento dinâmico sem restart e preservação de cache no boot | Validado |
| [02 — Thread Pool & Concorrência](02-tuning-thread-pool-concorrencia.md) | Plugin nativo de Thread Pool vs One-Thread-Per-Connection: eliminando context switching e saturação de RAM sob picos de conexões | Validado |
| [03 — Redo Log, I/O e Flushing](03-tuning-redo-log-io-flushing.md) | Redo log sizing, calibração de IOPS (`innodb_io_capacity`) e benchmark real de latência de commit (`flush_log_at_trx_commit` 1 vs 2) em Galera | Validado |
| [04 — Galera: Replicação, Concorrência e Flow Control](04-tuning-galera-concorrencia-flow-control.md) | Paralelismo de aplicação (`wsrep_slave_threads`), ajuste de Flow Control (`gcs.fc_limit`), prevenção de pausas no cluster e dimensionamento do Gcache | Validado |
| [05 — Plugins de Observabilidade: Query Response Time e MDL](05-tuning-plugins-observabilidade-query-response-time.md) | Profiling de baixa intrusão com o plugin `query_response_time` (histograma de latência p50/p95/p99) e inspeção de filas de locks com `metadata_lock_info` | Validado |
| [06 — Tabelas Temporárias (Aria) e Alocação Global de RAM](06-tuning-tabelas-temporarias-e-memoria-global.md) | Otimização de tabelas temporárias internas, motor Aria em substituição ao MyISAM, métricas de spill to disk e cálculo de teto contra o OOM Killer | Validado |

### Parte 2: Tuning Avançado de Queries & Otimizador de Custos
| Procedimento | Descrição | Status |
|---|---|---|
| [07 — Análise Profunda com EXPLAIN e ANALYZE JSON](07-tuning-queries-explain-analyze-json.md) | Telemetria real de execução: tempo na engine (`r_table_time_ms`), linhas lidas (`r_rows`), páginas acessadas no buffer e discrepâncias de catálogo | Validado |
| [08 — Índices Compostos, Leftmost Prefix e ICP](08-tuning-queries-indices-compostos-e-icp.md) | Regras da B-Tree, Index Condition Pushdown (`Using index condition`), consultas sargáveis e prevenção de supressão de índices por funções | Validado |
| [09 — Índices de Cobertura, Filesort e Paginação](09-tuning-queries-covering-index-e-ordenacao.md) | Estado de Covering Index (`Using index`), eliminação de ordenação em memória (`Using filesort`) e Keyset Pagination 450x mais rápida que `OFFSET` | Validado |
| [10 — Otimização de Joins, MRR e Optimizer Switch](10-tuning-queries-joins-mrr-e-optimizer-switch.md) | Tuning de junções relacionais, Multi-Range Read (transformando random I/O em linear), ordem de loops e calibração de flags no `optimizer_switch` | Validado |

---

## Diagrama da Metodologia de Tuning & Fluxo de Recursos

```
                             [ TRÁFEGO CLIENTE / VIP 192.168.1.100 ]
                                              │
                                              ▼
                      ┌───────────────────────────────────────────────┐
                      │    CAMADA 1: CONEXÕES & THREAD HANDLING       │
                      │  Thread Pool Nativo (pool-of-threads)          │
                      │  thread_pool_size = 4 | stall_limit = 500ms   │
                      │  Evita OOM e contenção de CPU por thread      │
                      └───────────────────────┬───────────────────────┘
                                              │
                                              ▼
                      ┌───────────────────────────────────────────────┐
                      │   CAMADA 2: OTIMIZADOR DE CONSULTAS & SQL     │
                      │  EXPLAIN & ANALYZE FORMAT=JSON (Telemetria)   │
                      │  Covering Index (Using index) & ICP           │
                      │  Eliminação de Using filesort & Keyset Paging │
                      │  Joins com Multi-Range Read (MRR)             │
                      └───────────────────────┬───────────────────────┘
                                              │
                                              ▼
                      ┌───────────────────────────────────────────────┐
                      │   CAMADA 3: EXECUÇÃO, PARSERS & TEMPORÁRIAS   │
                      │  tmp_table_size & max_heap_table_size (16MB)  │
                      │  Motor Aria em Spill to Disk (pagecache 128MB)│
                      │  sort_buffer / join_buffer calibrados         │
                      └───────────────────────┬───────────────────────┘
                                              │
                                              ▼
                      ┌───────────────────────────────────────────────┐
                      │      CAMADA 4: ENGINE INNODB & MEMÓRIA        │
                      │  InnoDB Buffer Pool (1GB lab / 70-80% RAM)   │
                      │  Hit Ratio > 99% | innodb_max_dirty_pages_pct │
                      │  Dynamic Resizing sem restart                 │
                      └───────────────┬───────────────┬───────────────┘
                                      │               │
                     Escrita / Commit │               │ Replicação Writeset
                                      ▼               ▼
        ┌───────────────────────────────────┐   ┌──────────────────────────────────┐
        │  CAMADA 5: I/O, REDO LOG & FLUSH  │   │   CAMADA 6: GALERA REPLICATION   │
        │  innodb_flush_log_at_trx_commit=2 │   │  wsrep_slave_threads = 8         │
        │  innodb_log_file_size (96MB)      │   │  gcs.fc_limit = 64 (anti-pausa)  │
        │  innodb_io_capacity = 200/2000    │   │  gcache.size = 512MB (anti-SST)  │
        └───────────────────────────────────┘   └──────────────────────────────────┘
                                      │               │
                                      ▼               ▼
                      ┌───────────────────────────────────────────────┐
                      │     CAMADA 7: OBSERVABILIDADE & PLUGINS       │
                      │  Plugin query_response_time (Histogramas)     │
                      │  Plugin metadata_lock_info (Detecção de MDL)  │
                      │  Agente PMM 3.10 (Exportação vmagent/mysqld)  │
                      └───────────────────────────────────────────────┘
```

---

## Mapa de Bolso do DBA (Onde Olhar / Comandos Essenciais)

| Recurso / Gargalo | O que checar | Comando essencial no MariaDB | Ação de Tuning Recomendada |
|---|---|---|---|
| **Buffer Pool Hit Ratio** | Taxa de leitura em cache vs disco | `SELECT ROUND((1 - (r.VARIABLE_VALUE / rq.VARIABLE_VALUE)) * 100, 2) FROM information_schema.GLOBAL_STATUS r JOIN information_schema.GLOBAL_STATUS rq ON rq.VARIABLE_NAME = 'Innodb_buffer_pool_read_requests' WHERE r.VARIABLE_NAME = 'Innodb_buffer_pool_reads';` | Se `< 99%`, aumentar `innodb_buffer_pool_size` dinamicamente. |
| **Saturação de Conexões** | Threads concorrentes e filas | `SELECT GROUP_ID, CONNECTIONS, THREADS, QUEUE_LENGTH, IS_STALLED FROM information_schema.THREAD_POOL_GROUPS;` | Habilitar `thread_handling=pool-of-threads` no `.cnf`. |
| **Diagnóstico Real de Query** | Tempo de engine e páginas lidas | `ANALYZE FORMAT=JSON <query>;` | Inspecionar `r_table_time_ms`, `r_rows` e `pages_accessed` para refinar índices. |
| **Varredura Completa Inútil** | Consulta ignorando índice | `EXPLAIN <query>;` | Verificar violação da *Leftmost Prefix Rule* ou funções no `WHERE` (tornar sargável). |
| **Ordenação em Disco/Memória** | Custo de CPU em ordenação | `EXPLAIN <query>;` procurando por `Using filesort` | Criar índice composto alinhando colunas do `WHERE` seguidas pelo `ORDER BY`. |
| **Paginação Lenta de API** | Páginas profundas com `OFFSET` | `ANALYZE FORMAT=JSON SELECT ... ORDER BY id LIMIT 90000, 20;` | Migrar para **Keyset Pagination** (`WHERE id > :last_id ORDER BY id LIMIT 20`). |
| **Contenção de I/O em Commit** | Fsyncs excessivos e waits | `SHOW GLOBAL STATUS LIKE 'Innodb_os_log_fsyncs';` e `SHOW GLOBAL STATUS LIKE 'Innodb_log_waits';` | Em cluster Galera, usar `innodb_flush_log_at_trx_commit = 2`. |
| **Pausa de Flow Control (Galera)** | Cluster congelando escritas | `SHOW GLOBAL STATUS LIKE 'wsrep_flow_control_paused';` | Aumentar `wsrep_slave_threads` para `2x` a `4x` vCPUs e calibrar `gcs.fc_limit`. |
| **Spill de Tabelas Temporárias** | Queries lentas usando disco | `SHOW GLOBAL STATUS LIKE 'Created_tmp_disk_tables';` | Aumentar `tmp_table_size` e `max_heap_table_size` sincronizados. |
| **Latência Oculta de Queries** | Distribuição temporal p50/p99 | `SELECT * FROM information_schema.QUERY_RESPONSE_TIME WHERE count > 0;` | Ativar o plugin `query_response_time` para análise de SLO/SLA. |

---

## Ambiente de Homologação

| Nó | IP | Função | CPU / RAM |
|---|---|---|---|
| **mariadb-db-01** | `192.168.1.21` | Nó de Dados Galera, HAProxy, Keepalived MASTER | 4 vCPUs / 7.4 GB RAM |
| **mariadb-db-02** | `192.168.1.23` | Nó de Dados Galera, HAProxy, Keepalived BACKUP | 4 vCPUs / 7.4 GB RAM |
| **mariadb-db-03** | `192.168.1.24` | Árbitro Galera (`garbd`), sem armazenamento | 2 vCPUs / 2.0 GB RAM |
| **VIP Lab** | `192.168.1.100` | IP Virtual Flutuante (Portas 3306 Escrita / 3307 Leitura) | Virtual IP (VRRP) |
| **pmm-mon-01** | `192.168.1.16` | PMM Server 3.10 (Docker, HTTPS 8443) | 4 vCPUs / 8.0 GB RAM |

- **SO:** Ubuntu 26.04.1 LTS
- **MariaDB Server:** 12.3.3-MariaDB-ubu2604-log
- **Galera Provider:** 26.4.27 (galera-4)

---

## Mantenedor

**DB Sentinel Consultoria em Tecnologia da Informação Ltda.**  
*Database Procedures Framework — Operational Standards & Best Practices.*
