# DB Sentinel | MariaDB — Observabilidade com Percona Monitoring and Management (PMM 3)

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração e sustentação de bancos de dados em ambientes corporativos de missão crítica.

Esta subpasta documenta a **Arquitetura de Observabilidade Centralizada com PMM 3.10** para MariaDB, cobrindo desde servidores *Single-Instance* (standalone) até a instrumentação avançada de **Clusters MariaDB Galera de Alta Disponibilidade** com balanceamento por HAProxy e Keepalived. Todos os procedimentos de instalação, permissões de menor privilégio, telemetria de queries (QAN), métricas de replicação síncrona e monitoramento de orquestradores foram **validados e homologados no laboratório real** integrado ao PMM Server (`https://192.168.1.16:8443`).

---

## Procedimentos

| Procedimento | Descrição | Status |
|---|---|---|
| [01 — Instalação e Arquitetura do PMM Client](01-observabilidade-pmm-instalacao-e-cliente.md) | Arquitetura do PMM 3.10, instalação do agente, portas de rede, autenticação segura e validação de conectividade | Validado |
| [02 — Monitoramento de Instância Single (Standalone)](02-observabilidade-mariadb-single-instance.md) | Criação de usuário de menor privilégio (`pmm`), ativação do Performance Schema, adição do serviço e Query Analytics (QAN) | Validado |
| [03 — Monitoramento de Alta Disponibilidade: Galera Cluster](03-observabilidade-galera-cluster-pmm.md) | Agrupamento de cluster (`mariadb_lab_cluster`), resolução de conflito de bind local, telemetria de nós de dados e tratamento de árbitros (`garbd`) | Validado |
| [04 — Monitoramento Dual: Orquestrador HAProxy e VIP](04-observabilidade-orquestrador-haproxy-dual-monitoring.md) | Monitoramento do balanceador com exportador nativo Prometheus (porta 8404), métricas de backends e agregação externa no PMM | Validado |
| [05 — Dashboards, Métricas-Chave e Melhores Práticas](05-observabilidade-dashboards-filtros-e-metricas-chave.md) | Navegação no Grafana do PMM 3, filtros corretos de instância (evitando serviços fantasmas), limiares de alerta e análise de QAN | Validado |

---

## Diagrama da Arquitetura de Observabilidade Dual

```
 ┌────────────────────────────────────────────────────────────────────────────────────────┐
 │                              PERCONA MONITORING & MANAGEMENT                           │
 │                               PMM SERVER 3.10 (192.168.1.16:8443)                      │
 │                                                                                        │
 │    ┌──────────────────────────┐   ┌────────────────────────┐   ┌──────────────────┐    │
 │    │    VictoriaMetrics (TSDB)│   │   ClickHouse (QAN Data)│   │  Grafana 11+ UI  │    │
 │    └─────────────▲────────────┘   └───────────▲────────────┘   └─────────▲────────┘    │
 └──────────────────┼────────────────────────────┼──────────────────────────┼─────────────┘
                    │ Push Metrics (443/8443)    │ Query Telemetry          │ Dashboards
                    │                            │                          │
 ┌──────────────────┴────────────────────────────┴──────────────────────────┴─────────────┐
 │ TOPOLOGIA DO LABORATÓRIO (Cluster Galera: mariadb_lab_cluster | Env: lab)              │
 │                                                                                        │
 │  ┌─────────────────────────────────┐           ┌─────────────────────────────────┐     │
 │  │      mariadb-db-01 (Dados)      │           │      mariadb-db-02 (Dados)      │     │
 │  │         192.168.1.21            │           │         192.168.1.23            │     │
 │  │                                 │           │                                 │     │
 │  │  • mariadbd (192.168.1.21:3306) │           │  • mariadbd (192.168.1.23:3306) │     │
 │  │  • HAProxy (VIP 192.168.1.100)  │           │  • HAProxy (Standby VIP)        │     │
 │  │                                 │           │                                 │     │
 │  │  [PMM AGENT]:                   │           │  [PMM AGENT]:                   │     │
 │  │   ├─ mysqld_exporter (42004)    │           │   ├─ mysqld_exporter (42004)    │     │
 │  │   ├─ mysql_perfschema_agent     │           │   ├─ mysql_perfschema_agent     │     │
 │  │   ├─ haproxy external (8404)    │           │   ├─ haproxy external (8404)    │     │
 │  │   ├─ node_exporter (42003)      │           │   ├─ node_exporter (42000)      │     │
 │  │   └─ vmagent (push metrics)     │           │   └─ vmagent (push metrics)     │     │
 │  └─────────────────▲───────────────┘           └─────────────────▲───────────────┘     │
 │                    │                                             │                     │
 │                    └──────────────────────┬──────────────────────┘                     │
 │                                           │ Multicast Replicação (4567)                │
 │                                           ▼                                            │
 │                        ┌─────────────────────────────────────┐                         │
 │                        │       mariadb-db-03 (Árbitro)       │                         │
 │                        │             192.168.1.24            │                         │
 │                        │                                     │                         │
 │                        │  • garbd (Galera Arbitrator Daemon) │                         │
 │                        │  [PMM AGENT]:                       │                         │
 │                        │   ├─ node_exporter (42000)          │                         │
 │                        │   └─ vmagent (push metrics)         │                         │
 │                        │   (Sem serviço falso de mysqld)     │                         │
 │                        └─────────────────────────────────────┘                         │
 └────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Mapa de Bolso do DBA (Onde Olhar / Comandos Essenciais)

| Recurso / Componente | O que inspecionar | Comando essencial / API | Ação Operacional |
|---|---|---|---|
| **Saúde dos Agentes Locais** | Status do pmm-agent e exporters | `pmm-admin status` | Todos os agentes devem reportar `Running` (nenhum em `Waiting` ou `Failed`). |
| **Serviços Ativos no PMM** | Lista de instâncias monitoradas | `pmm-admin list` | Valide o IP/porta real do nó (ex: `192.168.1.21:3306`), evitando `127.0.0.1`. |
| **Inventário Geral do PMM** | Todos os serviços (incluindo PG) | `pmm-admin inventory list services` | Garante que serviços de outros SGBDs (PostgreSQL/Patroni) estão preservados. |
| **Métrica do Cluster Galera** | Tamanho do quórum no Prometheus | `curl -sk -u admin:*** 'https://192.168.1.16:8443/prometheus/api/v1/query?query=mysql_global_status_wsrep_cluster_size'` | Deve retornar valor 3 para ambos os nós no cluster `mariadb_lab_cluster`. |
| **Métricas do HAProxy** | Conexões e status dos backends | `curl -sk -u admin:*** 'https://192.168.1.16:8443/prometheus/api/v1/query?query=haproxy_process_uptime_seconds'` | Valida se a camada de balanceamento está enviando métricas ao PMM. |
| **Filtro de Dashboards** | Evitar nós fantasmas no Grafana | No seletor `Service Name`: escolher `mariadb-db-01` ou `mariadb-db-02` | **Nunca** filtrar por `pmm-server-xxx` (instância interna do próprio container PMM). |

---

## Ambiente de Homologação

| Nó | IP | Papel no Lab | Agentes PMM Ativos |
|---|---|---|---|
| **pmm-mon-01** | `192.168.1.16` | PMM Server 3.10 (Docker, HTTPS 8443) | Servidor Central (VictoriaMetrics, ClickHouse, Grafana) |
| **mariadb-db-01** | `192.168.1.21` | Nó de Dados Galera / HAProxy MASTER | `mysqld_exporter`, `mysql_perfschema_agent`, `haproxy external`, `node_exporter` |
| **mariadb-db-02** | `192.168.1.23` | Nó de Dados Galera / HAProxy BACKUP | `mysqld_exporter`, `mysql_perfschema_agent`, `haproxy external`, `node_exporter` |
| **mariadb-db-03** | `192.168.1.24` | Árbitro Galera (`garbd`) | `node_exporter`, `vmagent` (sem mysqld) |
| **VIP Lab** | `192.168.1.100` | IP Virtual Flutuante (Portas 3306 / 3307) | Monitorado indiretamente via HAProxy |

- **SO:** Ubuntu 26.04.1 LTS
- **MariaDB Server:** 12.3.3-MariaDB-ubu2604-log
- **PMM Versão:** 3.10.0 (Client & Server)
- **Cluster Name:** `mariadb_lab_cluster` | **Environment:** `lab`

---

## Mantenedor

**DB Sentinel Consultoria em Tecnologia da Informação Ltda.**  
*Database Procedures Framework — Operational Standards & Best Practices.*
