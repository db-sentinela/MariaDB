# DB Sentinel | MariaDB Database Procedures Framework

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais padronizados (SOPs) para administração de bancos de dados corporativos em ambientes de missão crítica.

Este repositório cobre **MariaDB 12** em topologia de alta disponibilidade com **Galera Cluster, HAProxy e Keepalived**, além de estratégias de troubleshooting, backup/restore e tuning avançado. Todos os procedimentos são independentes de clientes, não contêm dados de ambientes reais e foram rigorosamente **executados, testados e homologados em laboratório real** antes da publicação.

---

## Módulos de Conhecimento

### 1. [Galera HA, Alta Disponibilidade e Operação](Galera/README.md)
| Procedimento | Descrição | Status |
|---|---|---|
| [Galera HA com HAProxy e Keepalived](Galera/galera-ha-haproxy-keepalived.md) | Topologia e instalação: cluster 2 nós + árbitro, IP virtual único (3306 escrita / 3307 leitura) | Validado |
| [Testes de falha e failover](Galera/galera-testes-de-falha-failover.md) | Queda de nó de dados, failover de escrita, migração do VIP e queda do árbitro | Validado |
| [Recuperação de queda total](Galera/galera-recuperacao-queda-total.md) | Cluster todo parado: diagnóstico, escolha do nó e bootstrap seguro (`safe_to_bootstrap`) | Validado |
| [Perda de quórum e recuperação](Galera/galera-perda-de-quorum-recuperacao.md) | Partição de rede, proteção contra split-brain e volta ao estado `Primary` | Validado |
| [Monitoramento e observabilidade](Galera/galera-monitoramento-observabilidade.md) | Variáveis `wsrep`, flow control, estatísticas do HAProxy, health check e alertas | Validado |
| [Operação do dia a dia](Galera/galera-operacao-dia-a-dia.md) | Rolling restart sem downtime, backup com `mariadb-backup`, adicionar/remover nós | Validado |

### 2. [Troubleshooting & Diagnóstico de Incidentes](Troubleshooting/README.md)
| Procedimento | Descrição | Status |
|---|---|---|
| [Locks e bloqueios](Troubleshooting/troubleshooting-locks-e-bloqueios.md) | Identificar quem bloqueia quem (row lock e metadata lock), resolução com `KILL` e comportamento de BF-abort em Galera | Validado |
| [Deadlocks](Troubleshooting/troubleshooting-deadlocks.md) | Mecânica do deadlock, leitura do relatório `LATEST DETECTED DEADLOCK`, registro em log e prevenção | Validado |
| [Diagnóstico — onde procurar](Troubleshooting/troubleshooting-diagnostico-onde-procurar.md) | Mapa de "pergunta → onde olhar": processlist, `INNODB_TRX`, lock waits, logs e `sys` schema | Validado |
| [Queries lentas](Troubleshooting/troubleshooting-queries-lentas.md) | Slow log, `EXPLAIN`/`ANALYZE` e otimização por índices (full table scan → ref) | Validado |
| [Problemas de conexão](Troubleshooting/troubleshooting-conexoes.md) | Erro `Too many connections` (1040): diagnóstico de saturação, vaga reservada de `SUPER` e conexões órfãs | Validado |
| [Galera: nó não entra (SST/IST)](Troubleshooting/troubleshooting-galera-sst-join.md) | State transfer: diferenças entre IST e SST, SST completo forçado e resolução de falhas de join | Validado |

### 3. [Backup & Restore (Estratégia e Recuperação)](Backup&Restore/README.md)
| Procedimento | Descrição | Status |
|---|---|---|
| [Visão geral e estratégia](Backup&Restore/backup-restore-visao-geral-estrategia.md) | Políticas de RPO/RTO, ciclo de vida de backups e arquitetura de retenção | Validado |
| [Backup lógico com mariadb-dump](Backup&Restore/backup-restore-logico-mariadb-dump.md) | Dump consistente com `single-transaction`, restore granular e exportação de objetos | Validado |
| [Backup físico com mariadb-backup](Backup&Restore/backup-restore-fisico-mariadb-backup.md) | Backup full e incremental sem parada, fase de `prepare` e restauração física | Validado |
| [PITR via Binary Log](Backup&Restore/backup-restore-pitr-binlog.md) | Point-in-Time Recovery: replay de binlogs até o segundo exato anterior a desastres | Validado |
| [Cenários de recuperação de desastres](Backup&Restore/backup-restore-cenarios-recuperacao.md) | Playbook passo a passo para perda de tabela, corrupção de partição e falha de storage | Validado |

### 4. [Tuning & Otimização de Performance](Tuning/README.md)
| Procedimento | Descrição | Status |
|---|---|---|
| [01 — InnoDB Buffer Pool & Memória](Tuning/01-tuning-innodb-buffer-pool-memoria.md) | Dimensionamento, Hit Ratio real, Dirty Pages flushing, resize dinâmico e dump/load de cache | Validado |
| [02 — Thread Pool & Concorrência](Tuning/02-tuning-thread-pool-concorrencia.md) | Plugin nativo de Thread Pool vs One-Thread-Per-Connection: escalabilidade sob milhares de conexões | Validado |
| [03 — Redo Log, I/O e Flushing](Tuning/03-tuning-redo-log-io-flushing.md) | Redo log sizing, calibração de IOPS e benchmark real de latência (`innodb_flush_log_at_trx_commit` 1 vs 2) | Validado |
| [04 — Galera: Replicação e Flow Control](Tuning/04-tuning-galera-concorrencia-flow-control.md) | Paralelismo (`wsrep_slave_threads`), ajuste de Flow Control (`gcs.fc_limit`) e dimensionamento de Gcache | Validado |
| [05 — Plugins de Observabilidade (QRT e MDL)](Tuning/05-tuning-plugins-observabilidade-query-response-time.md) | Profiling de baixa intrusão com `query_response_time` (p50/p95/p99) e detecção de filas de metadados | Validado |
| [06 — Tabelas Temporárias (Aria) e RAM Global](Tuning/06-tuning-tabelas-temporarias-e-memoria-global.md) | Tabelas temporárias internas, motor Aria em substituição ao MyISAM e fórmula contra o OOM Killer | Validado |

---

## Ambiente de Homologação

| Item | Especificação do Laboratório |
|---|---|
| **Sistema Operacional** | Ubuntu 26.04.1 LTS |
| **MariaDB Server** | 12.3.3-MariaDB-ubu2604-log (mariadb.org binary distribution) |
| **Galera Cluster** | galera-4 / garbd 26.4.27 |
| **Balanceador & VIP** | HAProxy 2.8+ / Keepalived (VRRP na porta 3306 escrita / 3307 leitura) |
| **Observabilidade** | Percona Monitoring and Management (PMM Server 3.10.0) |
| **Virtualização** | Oracle VirtualBox (Ambiente de redes privadas isoladas) |

---

## Estrutura do Repositório

```
MariaDB/
├── README.md
├── Galera/
│   ├── README.md
│   ├── galera-ha-haproxy-keepalived.md
│   ├── galera-testes-de-falha-failover.md
│   ├── galera-recuperacao-queda-total.md
│   ├── galera-perda-de-quorum-recuperacao.md
│   ├── galera-monitoramento-observabilidade.md
│   └── galera-operacao-dia-a-dia.md
├── Troubleshooting/
│   ├── README.md
│   ├── troubleshooting-locks-e-bloqueios.md
│   ├── troubleshooting-deadlocks.md
│   ├── troubleshooting-diagnostico-onde-procurar.md
│   ├── troubleshooting-queries-lentas.md
│   ├── troubleshooting-conexoes.md
│   └── troubleshooting-galera-sst-join.md
├── Backup&Restore/
│   ├── README.md
│   ├── backup-restore-visao-geral-estrategia.md
│   ├── backup-restore-logico-mariadb-dump.md
│   ├── backup-restore-fisico-mariadb-backup.md
│   ├── backup-restore-pitr-binlog.md
│   └── backup-restore-cenarios-recuperacao.md
└── Tuning/
    ├── README.md
    ├── 01-tuning-innodb-buffer-pool-memoria.md
    ├── 02-tuning-thread-pool-concorrencia.md
    ├── 03-tuning-redo-log-io-flushing.md
    ├── 04-tuning-galera-concorrencia-flow-control.md
    ├── 05-tuning-plugins-observabilidade-query-response-time.md
    └── 06-tuning-tabelas-temporarias-e-memoria-global.md
```

---

## Mantenedor

**DB Sentinel Consultoria em Tecnologia da Informação Ltda.**  
*Database Procedures Framework — Operational Standards & Best Practices.*
