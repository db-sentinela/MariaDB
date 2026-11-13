# 03 — Monitoramento de Alta Disponibilidade: Galera Cluster no PMM

Quando nós MariaDB deixam de operar como instâncias isoladas (*Single-Instance*) e passam a formar um **Galera Cluster de Alta Disponibilidade**, o modelo de monitoramento precisa ser completamente reestruturado. É necessário correlacionar os nós sob o mesmo identificador de cluster, monitorar o quórum e a replicação síncrona, resolver os conflitos de rede causados por balanceadores locais e evitar armadilhas comuns com nós árbitros (**garbd**).

---

## O que é

No PMM 3, o monitoramento de um Galera Cluster agrega três componentes vitais:
1. **Identificação e Agrupamento Lógico (`--cluster`):** Ao adicionar o serviço em cada nó de dados (`mariadb-db-01` e `mariadb-db-02`), a flag `--cluster=mariadb_lab_cluster` instrui o PMM Server a unificar as séries temporais no dashboard oficial **"MySQL Galera Summary"**.
2. **Coleta de Métricas `wsrep_*`:** O `mysqld_exporter` extrai automaticamente mais de 50 contadores críticos do Galera, incluindo:
   - Tamanho do cluster (`wsrep_cluster_size`).
   - Estado de sincronização (`wsrep_local_state_comment`).
   - Controles de fluxo e pausas (`wsrep_flow_control_paused`).
   - Filas de entrada e saída (`wsrep_local_recv_queue`, `wsrep_local_send_queue`).
   - Conflitos de certificação e abortos (`wsrep_local_cert_failures`, `wsrep_local_bf_aborts`).
3. **Tratamento Diferenciado de Nós Árbitros (`garbd`):** O árbitro Galera (`mariadb-db-03`) participa exclusivamente do quórum de votação e **não possui banco de dados local ou tabelas**. Logo, ele não deve ter serviço de MySQL registrado no PMM, mas sim monitoramento de host (`node_exporter`).

---

## Por que importa

1. **Visibilidade de Quórum e Proteção contra Split-Brain:** Se a rede entre os nós oscilar, o PMM detecta a queda do tamanho do cluster (`wsrep_cluster_size < 3`) e aponta se os nós entraram em estado `Non-Primary`, permitindo rápida intervenção antes que a aplicação sofra indisponibilidade.
2. **Diagnóstico Imediato de Flow Control:** Quando um nó do cluster sofre degradação de hardware ou acúmulo de writesets, o Galera aciona pausas síncronas em todos os outros nós. O dashboard de Galera do PMM expõe o nó exato que está estrangulando o cluster.
3. **Evita Alarmes Falsos e Erros de Conexão Recusada:** Registrar serviços de MySQL apontando para `127.0.0.1` em ambientes onde o banco escuta no IP privado (para conviver com o HAProxy) gera milhares de erros de `Connection refused` nos logs e alertas falsos de banco fora do ar.

---

## Como reproduzir

Execute os passos a seguir no laboratório para configurar a monitoração do cluster Galera:

### 1. Garantir que o usuário `pmm` possui permissões na rede privada
No nó mestre (`mariadb-db-01`), execute no MariaDB:
```sql
-- Cria o usuario pmm permitindo conexao a partir de qualquer host da rede privada
CREATE USER IF NOT EXISTS 'pmm'@'192.168.1.%' IDENTIFIED BY 'Pmm#2026' WITH MAX_USER_CONNECTIONS 10;
ALTER USER 'pmm'@'192.168.1.%' IDENTIFIED BY 'Pmm#2026';

GRANT SELECT, PROCESS, REPLICATION CLIENT, RELOAD, BINLOG MONITOR ON *.* TO 'pmm'@'192.168.1.%';
GRANT SELECT ON performance_schema.* TO 'pmm'@'192.168.1.%';
FLUSH PRIVILEGES;
```
*(As instruções DDL e GRANT são replicadas automaticamente para o `mariadb-db-02` pelo Galera).*

### 2. A Grande Pegadinha do Bind Address e Como Registrar no PMM
Como o nó possui HAProxy escutando no VIP `192.168.1.100:3306`, o MariaDB está vinculado estritamente ao IP da interface física (`192.168.1.21` no db-01 e `192.168.1.23` no db-02). 

**No Nó `mariadb-db-01` (192.168.1.21):**
```bash
# Se houver servico anterior mal configurado, remova-o
sudo pmm-admin remove mysql mariadb-01 2>/dev/null || true

# Registre informando explicitamente o IP do no e o nome do cluster
sudo pmm-admin add mysql \
  --service-name=mariadb-db-01 \
  --host=192.168.1.21 \
  --port=3306 \
  --username=pmm \
  --password='Pmm#2026' \
  --cluster=mariadb_lab_cluster \
  --query-source=perfschema \
  --environment=lab
```

**No Nó `mariadb-db-02` (192.168.1.23):**
```bash
sudo pmm-admin remove mysql mariadb-db-02 2>/dev/null || true

sudo pmm-admin add mysql \
  --service-name=mariadb-db-02 \
  --host=192.168.1.23 \
  --port=3306 \
  --username=pmm \
  --password='Pmm#2026' \
  --cluster=mariadb_lab_cluster \
  --query-source=perfschema \
  --environment=lab
```

### 3. Limpeza Operacional no Nó Árbitro (`mariadb-db-03`)
No nó `mariadb-db-03` (192.168.1.24), certifique-se de que nenhum serviço de MySQL fantasma esteja cadastrado:
```bash
# Remove servico MySQL inativo no arbitro garbd
sudo pmm-admin remove mysql mariadb-db-03 2>/dev/null || true

# Valide que apenas node_exporter e vmagent estao ativos
sudo pmm-admin list
```

### 4. Validar as métricas de Galera na API do PMM Server
Em qualquer um dos nós, consulte o VictoriaMetrics do PMM Server para confirmar a recepção de métricas do cluster:
```bash
curl -sk -u admin:Admin#2026 \
  'https://192.168.1.16:8443/prometheus/api/v1/query?query=mysql_global_status_wsrep_cluster_size'
```

---

## O que você vê (saída real)

### 1. Adição com sucesso no `mariadb-db-01`:
```text
MySQL Service added.
Service ID  : 8814a8f7-3458-41d9-84c6-de8dc06343ff
Service name: mariadb-db-01

Table statistics collection enabled (the limit is 1000, the actual table count is 312).
```

### 2. Status consolidado do `mariadb-db-01`:
```text
Service type        Service name         Address and port         Service ID
MySQL               mariadb-db-01        192.168.1.21:3306        8814a8f7-3458-41d9-84c6-de8dc06343ff

Agent type                    Status           Metrics Mode        Agent ID                                    Service ID                                  Port
pmm_agent                     Connected                            4dc4d1b6-bf21-44e3-a270-f13fbdc8da82                                                    0 
node_exporter                 Running          push                b4c61a0a-15a8-43ac-abbf-a60e16a11616                                                    42003 
mysqld_exporter               Running          push                6a3e2bd6-92b3-4f60-b8c5-8eb85c5ac5fc        8814a8f7-3458-41d9-84c6-de8dc06343ff        42004 
mysql_perfschema_agent        Running                              8875f93b-87f3-47e2-a796-9375806fef16        8814a8f7-3458-41d9-84c6-de8dc06343ff        0 
vmagent                       Running          push                9b40c7dd-634a-42b8-90f8-5549090ad6ab                                                    42002 
nomad_agent                   Running                              0145353b-9157-4a73-bde5-77ac1daf1b92                                                    42000
```

### 3. Consulta em tempo real das métricas do cluster no PMM Server:
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
          "environment": "lab",
          "node_name": "mariadb-db-01",
          "service_name": "mariadb-db-01",
          "service_type": "mysql"
        },
        "value": [1794609087, "3"]
      },
      {
        "metric": {
          "__name__": "mysql_global_status_wsrep_cluster_size",
          "cluster": "mariadb_lab_cluster",
          "environment": "lab",
          "node_name": "mariadb-db-02",
          "service_name": "mariadb-db-02",
          "service_type": "mysql"
        },
        "value": [1794609087, "3"]
      }
    ]
  }
}
```
**Análise das Evidências Reais:**
- Ambos os nós (`mariadb-db-01` e `mariadb-db-02`) reportam tamanho de cluster = `3`.
- Ambos estão associados à tag de agrupamento `"cluster": "mariadb_lab_cluster"`.
- A coleta opera em modo síncrono contínuo com resolução de segundo.

---

## Lições

- **Sempre Padronize os Nomes de Serviço:** Evite inconsistências como nomear um nó como `mariadb-01` e o outro como `mariadb-db-02`. A convenção do framework DB Sentinel exige prefixos uniformes (`mariadb-db-01`, `mariadb-db-02`).
- **O Papel da Tag `--cluster`:** Se você não passar `--cluster=mariadb_lab_cluster`, o PMM tratará os nós como bancos standalone separados. Os gráficos de quórum, replicação cruzada e divergência de seqno não serão renderizados.
- **Isolamento de PostgreSQL / Patroni:** Conforme comprovado via `pmm-admin inventory list services`, operações de adição ou remoção de serviços MySQL no cliente de banco **não afetam** os outros clusters cadastrados no PMM (como PostgreSQL/Patroni), pois cada nó gerencia estritamente o seu próprio `Node ID`.

---

## Pegadinhas

- **O Erro `connect: connection refused` no `127.0.0.1:3306`:** Se o PMM Client tentar conectar no localhost quando o MariaDB estiver em bind address dedicado (`bind-address = 192.168.1.21`), o exporter falhará com erro `dial tcp 127.0.0.1:3306: connect: connection refused`. Sempre passe explicitamente o IP do nó com `--host=<ip_do_no>`.
- **Serviço Fantasma de MySQL no Árbitro (`db-03`):** Cadastrar um serviço de banco no árbitro é um erro clássico de estagiários. O árbitro não roda `mariadbd` (porta 3306 fechada), fazendo o PMM alertar continuamente que a instância está fora do ar. Mantenha no árbitro apenas o `node_exporter`.
- **Reinstalação sem `--service-name` Explícito:** Se você rodar `pmm-admin add mysql` sem especificar `--service-name`, o PMM adotará o hostname padrão do sistema operacional, o que pode gerar colisões caso múltiplos nós tenham o mesmo hostname base.
