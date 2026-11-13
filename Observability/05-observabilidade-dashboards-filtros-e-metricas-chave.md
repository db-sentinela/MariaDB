# 05 — Dashboards do PMM, Métricas-Chave do Galera e Boas Práticas de Filtros

Visualizar métricas no Grafana sem compreender as variáveis de agregação e as peculiaridades do PMM Server leva a interpretações falsas, alarmes indevidos e tempo perdido durante incidentes críticos. Este documento ensina como navegar nos painéis oficiais do **PMM 3.10**, configurar os filtros de cluster e serviço corretamente (evitando nós fantasmas) e interpretar as métricas de saúde do MariaDB Galera Cluster e do **Query Analytics (QAN)**.

---

## O que é

O PMM disponibiliza uma biblioteca completa de dashboards pré-construídos para MariaDB e Galera:
1. **MySQL Galera Summary:** Painel focado na saúde global do cluster — tamanho do quórum (`wsrep_cluster_size`), estado dos nós (`Synced`, `Donor`, `Joined`), taxas de controle de fluxo e tamanho de filas.
2. **MySQL Instance Summary:** Painel focado na saúde da instância individual — CPU, consumo do Buffer Pool, threads ativas, taxa de commits, I/O e queries por segundo (QPS).
3. **PMM Query Analytics (QAN):** Módulo especializado que analisa o perfil de carga do banco em tempo real, agrupando consultas semelhantes em *Query Digests* e ordenando-as por tempo de carga total (*Query Load*).

---

## Por que importa

1. **Eliminação do Erro do "Nó Fantasma" (`pmm-server-xxx`):** O PMM Server roda dentro de um container Docker que possui sua própria instância interna de banco de dados (`pmm-server-postgresql` ou similar). Por padrão de ordenação alfabética, o Grafana frequentemente pré-seleciona essa instância interna no filtro. Se o DBA não souber trocar o filtro, ele analisará os dados do próprio servidor de monitoramento acreditando que é o seu banco de produção!
2. **Diagnóstico Rápido em Janelas de Crise:** Durante um incidente, saber exatamente em quais 3 painéis olhar reduz o MTTR (*Mean Time To Resolution*) de 30 minutos para 2 minutos.
3. **Identificação de Queries Vilãs no QAN:** O QAN permite correlacionar picos de CPU com a consulta exata responsável pela carga, exibindo a query parametrizada, o percentil p95 de latência e a contagem de execuções.

---

## Como reproduzir

Acesse a interface web do PMM Server no navegador ou audite as séries via API:

### 1. Acesso à Interface Web do PMM
- **URL:** `https://192.168.1.16:8443`
- **Usuário:** `admin`
- **Senha:** `Admin#2026`

### 2. Configurar os Filtros de Variáveis no Topo do Dashboard
Ao abrir o dashboard **"MySQL Galera Summary"** ou **"MySQL Instance Summary"**, ajuste os seletores na barra superior:
- **Environment:** Selecione `lab`
- **Cluster:** Selecione `mariadb_lab_cluster`
- **Service Name:** Selecione `mariadb-db-01` ou `mariadb-db-02`
- **Interval:** Ajuste para `1m` ou `5m`

### 3. Consultar as Métricas Essenciais de Galera via API
Para scripts de automação ou checagens rápidas via terminal sem abrir a interface gráfica:

```bash
# 3A: Checar o Quorum do Cluster (Deve ser 3)
curl -sk -u admin:Admin#2026 \
  'https://192.168.1.16:8443/prometheus/api/v1/query?query=mysql_global_status_wsrep_cluster_size'

# 3B: Checar o Estado dos Nos (Deve ser 4 = Synced)
curl -sk -u admin:Admin#2026 \
  'https://192.168.1.16:8443/prometheus/api/v1/query?query=mysql_global_status_wsrep_local_state'

# 3C: Checar Taxa de Flow Control (Pausa do Cluster)
curl -sk -u admin:Admin#2026 \
  'https://192.168.1.16:8443/prometheus/api/v1/query?query=rate(mysql_global_status_wsrep_flow_control_paused[5m])'
```

---

## O que você vê (saída real)

### 1. Métrica de Quórum capturada no VictoriaMetrics do PMM Server:
```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "__name__": "mysql_global_status_wsrep_cluster_size",
          "cluster": "mariadb_lab_cluster",
          "node_name": "mariadb-db-01",
          "service_name": "mariadb-db-01"
        },
        "value": [1794609087, "3"]
      },
      {
        "metric": {
          "__name__": "mysql_global_status_wsrep_cluster_size",
          "cluster": "mariadb_lab_cluster",
          "node_name": "mariadb-db-02",
          "service_name": "mariadb-db-02"
        },
        "value": [1794609087, "3"]
      }
    ]
  }
}
```

### 2. Métrica de Estado Local dos Nós (`wsrep_local_state`):
```text
mariadb-db-01: 4
mariadb-db-02: 4
```
*(No protocolo Galera, o estado 4 corresponde a `Synced`, significando que o nó está 100% alinhado com o grupo e pronto para ler e escrever).*

### 3. Tabela de Estados Numéricos do Galera:
| Código Numérico | Estado no Galera | Significado Operacional |
|---|---|---|
| **1** | `Joining` | Nó negociando entrada no cluster |
| **2** | `Donor/Desynced` | Nó atuando como doador de estado para outro nó |
| **3** | `Joined` | Nó aplicando writesets pendentes para sincronizar |
| **4** | `Synced` | **Saudável:** Nó totalmente sincronizado e operacional |

---

## Lições

- **Matriz de Alerta do Galera no PMM:**
  - `mysql_global_status_wsrep_cluster_size < 3`: Alerta imediato de severidade ALTA (perda de nó ou partição de rede).
  - `mysql_global_status_wsrep_local_state != 4`: Nó desincronizado (SST ou falha de rede).
  - `rate(mysql_global_status_wsrep_flow_control_paused[1m]) > 0.05`: Alerta de performance (o cluster está passando mais de 5% do tempo pausado).
  - `mysql_global_status_wsrep_local_recv_queue_avg > 1.0`: Nós de aplicação não estão conseguindo processar os writesets na mesma velocidade que chegam.
- **Como Interpretar o QAN:** No Query Analytics, foque na coluna **Load %**. Uma consulta que roda em 1 ms mas é executada 500.000 vezes por minuto gera muito mais carga total no banco do que uma consulta lenta que roda apenas uma vez por dia.

---

## Pegadinhas

- **A Armadilha do `pmm-server-xxx` no Filtro do Grafana:** Nunca abra um chamado ou analise um gráfico sem verificar o seletor `Service Name`. Se o filtro estiver em `pmm-server-postgresql` ou similar, você verá conexões e uso de CPU do próprio servidor PMM, e não do seu cluster MariaDB.
- **Alertar sobre `wsrep_flow_control_paused` Absoluto:** Esse contador do MySQL é cumulativo desde o boot. Alertar sobre `wsrep_flow_control_paused > 0` fará o monitoramento disparar para sempre após a primeira pausa momentânea ocorrida semanas atrás. A regra de alerta deve sempre usar a função **`rate()`** (ex: `rate(wsrep_flow_control_paused[5m]) > 0.01`).
- **Nó Árbitro sem Gráfico de Banco:** O nó `mariadb-db-03` aparecerá apenas nos dashboards de sistema operacional ("Node Summary"), pois não executa a engine MariaDB. Não tente buscar métricas de InnoDB para ele.
