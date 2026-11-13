# DB Sentinel | MariaDB — Tuning & Otimização de Performance

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração de bancos de dados em ambientes de missão crítica.

Esta subpasta cobre o **Tuning e Otimização Avançada do MariaDB 12** em topologia de alta disponibilidade (Cluster Galera com 2 nós de dados, árbitro garbd, HAProxy e Keepalived). Todos os procedimentos, fórmulas matemáticas de dimensionamento, parâmetros de engines, ajustes dinâmicos e plugins de observabilidade foram **executados, mensurados e validados no laboratório real** antes da consolidação desta documentação.

---

## Procedimentos

| Procedimento | Descrição | Status |
|---|---|---|
| [01 — InnoDB Buffer Pool & Memória](01-tuning-innodb-buffer-pool-memoria.md) | Dimensionamento, Hit Ratio real, Dirty Pages flushing, redimensionamento dinâmico sem restart e preservação de cache no boot | Validado |
| [02 — Thread Pool & Concorrência](02-tuning-thread-pool-concorrencia.md) | Plugin nativo de Thread Pool vs One-Thread-Per-Connection: eliminando context switching e saturação de RAM sob picos de conexões | Validado |
| [03 — Redo Log, I/O e Flushing](03-tuning-redo-log-io-flushing.md) | Redo log sizing, calibração de IOPS (`innodb_io_capacity`) e benchmark real de latência de commit (`flush_log_at_trx_commit` 1 vs 2) em Galera | Validado |
| [04 — Galera: Replicação, Concorrência e Flow Control](04-tuning-galera-concorrencia-flow-control.md) | Paralelismo de aplicação (`wsrep_slave_threads`), ajuste de Flow Control (`gcs.fc_limit`), prevenção de pausas no cluster e dimensionamento do Gcache | Validado |
| [05 — Plugins de Observabilidade: Query Response Time e MDL](05-tuning-plugins-observabilidade-query-response-time.md) | Profiling de baixa intrusão com o plugin `query_response_time` (histograma de latência p50/p95/p99) e inspeção de filas de locks com `metadata_lock_info` | Validado |
| [06 — Tabelas Temporárias (Aria) e Alocação Global de RAM](06-tuning-tabelas-temporarias-e-memoria-global.md) | Otimização de tabelas temporárias internas, motor Aria em substituição ao MyISAM, métricas de spill to disk e cálculo de teto contra o OOM Killer | Validado |

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
                      │   CAMADA 2: EXECUÇÃO, PARSERS & TEMPORÁRIAS   │
                      │  tmp_table_size & max_heap_table_size (16MB)  │
                      │  Motor Aria em Spill to Disk (pagecache 128MB)│
                      │  sort_buffer / join_buffer calibrados         │
                      └───────────────────────┬───────────────────────┘
                                              │
                                              ▼
                      ┌───────────────────────────────────────────────┐
                      │      CAMADA 3: ENGINE INNODB & MEMÓRIA        │
                      │  InnoDB Buffer Pool (1GB lab / 70-80% RAM)   │
                      │  Hit Ratio > 99% | innodb_max_dirty_pages_pct │
                      │  Dynamic Resizing sem restart                 │
                      └───────────────┬───────────────┬───────────────┘
                                      │               │
                     Escrita / Commit │               │ Replicação Writeset
                                      ▼               ▼
        ┌───────────────────────────────────┐   ┌──────────────────────────────────┐
        │  CAMADA 4: I/O, REDO LOG & FLUSH  │   │   CAMADA 5: GALERA REPLICATION   │
        │  innodb_flush_log_at_trx_commit=2 │   │  wsrep_slave_threads = 8         │
        │  innodb_log_file_size (96MB)      │   │  gcs.fc_limit = 64 (anti-pausa)  │
        │  innodb_io_capacity = 200/2000    │   │  gcache.size = 512MB (anti-SST)  │
        └───────────────────────────────────┘   └──────────────────────────────────┘
                                      │               │
                                      ▼               ▼
                      ┌───────────────────────────────────────────────┐
                      │     CAMADA 6: OBSERVABILIDADE & PLUGINS       │
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
