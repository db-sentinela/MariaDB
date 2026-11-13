# 04 — Monitoramento Dual: Orquestrador HAProxy e VIP no PMM

Em arquiteturas empresariais de Alta Disponibilidade (HA), o banco de dados nunca opera sozinho: ele depende de uma camada de proxy ou balanceador de carga (**HAProxy**) e de um roteador de IP flutuante (**Keepalived / VIP**). O padrão **Monitoramento Dual (Engine + Orquestrador)** preconiza que a telemetria do balanceador deve ser coletada e correlacionada dentro da mesma plataforma de observabilidade (PMM 3), permitindo diagnosticar se uma degradação de serviço é originada no banco ou no proxy.

---

## O que é

No laboratório MariaDB Galera:
- O tráfego de escrita da aplicação conecta no **VIP `192.168.1.100:3306`**.
- O balanceador **HAProxy** escuta na porta 3306, valida a saúde do Galera via HTTP na porta 9200 (`httpchk`) e encaminha as conexões para o nó ativo (`mariadb-db-01`), mantendo o nó secundário (`mariadb-db-02`) em standby.
- O HAProxy 2.8+ possui um exportador nativo de métricas Prometheus embutido (`http-request use-service prometheus-exporter`).
- Através do comando **`pmm-admin add external`**, o PMM 3 ingere essas métricas nativas diretamente na porta de estatísticas (8404), integrando contadores de conexões frontend, saturação de sessões, latência de checagem e estado dos backends (`UP`/`DOWN`).

---

## Por que importa

1. **Visibilidade do Caminho Completo da Aplicação:** Uma aplicação pode receber erro `Connection timed out` mesmo com o MariaDB perfeitamente saudável se o HAProxy tiver atingido seu limite de `maxconn` (4096 conexões) ou se o healthcheck na porta 9200 tiver oscilado. O monitoramento dual expõe o gargalo na camada exata.
2. **Detecção de Failover do VIP:** O HAProxy registra imediatamente quando um backend é desativado ou entra em estado `DOWN`, alertando o time de sustentação no momento exato em que a escrita migra para o nó reserva.
3. **Correlação de Tráfego:** Permite cruzar a taxa de conexões recebidas pelo balanceador com a taxa de queries executadas no banco, identificando conexões órfãs ou aplicações abrindo conexões sem efetuar transações.

---

## Como reproduzir

Execute os passos a seguir nos nós que hospedam o HAProxy (`mariadb-db-01` e `mariadb-db-02`):

### 1. Habilitar o Exportador Nativo Prometheus no HAProxy
Edite o arquivo `/etc/haproxy/haproxy.cfg` e adicione a diretiva `prometheus-exporter` no bloco `listen stats`:

```ini
listen stats
    mode http
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 5s
    # Habilita o endpoint nativo Prometheus na URI /metrics
    http-request use-service prometheus-exporter if { path /metrics }
```

Recarregue o serviço sem derrubar conexões ativas:
```bash
sudo systemctl reload haproxy
```

### 2. Validar o endpoint de métricas localmente
```bash
curl -s http://127.0.0.1:8404/metrics | grep "haproxy_process_uptime_seconds"
```

### 3. Adicionar o HAProxy como Serviço Externo no PMM Client
No nó `mariadb-db-01`:
```bash
sudo pmm-admin add external \
  --service-name=haproxy-mariadb-01 \
  --listen-port=8404 \
  --metrics-path=/metrics \
  --environment=lab \
  --cluster=mariadb_lab_cluster
```

No nó `mariadb-db-02`:
```bash
sudo pmm-admin add external \
  --service-name=haproxy-mariadb-02 \
  --listen-port=8404 \
  --metrics-path=/metrics \
  --environment=lab \
  --cluster=mariadb_lab_cluster
```

### 4. Validar o recebimento das métricas no PMM Server
Consulte a API do VictoriaMetrics no servidor central:
```bash
curl -sk -u admin:Admin#2026 \
  'https://192.168.1.16:8443/prometheus/api/v1/query?query=haproxy_process_uptime_seconds'
```

---

## O que você vê (saída real)

### 1. Adição do exportador externo confirmada no terminal:
```text
External Service added.
Service ID  : d56db1dd-97e8-4d77-bc6d-5307c08f2aa4
Service name: haproxy-mariadb-01
Group       : external
```

### 2. Status dos serviços locais no nó `mariadb-db-01`:
```text
Service type             Service name              Address and port         Service ID
MySQL                    mariadb-db-01             192.168.1.21:3306        8814a8f7-3458-41d9-84c6-de8dc06343ff
External:external        haproxy-mariadb-01                                 d56db1dd-97e8-4d77-bc6d-5307c08f2aa4

Agent type                    Status           Metrics Mode        Agent ID                                    Service ID                                  Port
node_exporter                 Running          push                b4c61a0a-15a8-43ac-abbf-a60e16a11616                                                    42003 
mysqld_exporter               Running          push                6a3e2bd6-92b3-4f60-b8c5-8eb85c5ac5fc        8814a8f7-3458-41d9-84c6-de8dc06343ff        42004 
mysql_perfschema_agent        Running                              8875f93b-87f3-47e2-a796-9375806fef16        8814a8f7-3458-41d9-84c6-de8dc06343ff        0 
external-exporter             Running          push                e42be43b-286a-4c2a-a983-9b993df46084        d56db1dd-97e8-4d77-bc6d-5307c08f2aa4        8404 
```

### 3. Resposta em tempo real da série temporal no PMM Server:
```json
{
  "status": "success",
  "data": {
    "resultType": "vector",
    "result": [
      {
        "metric": {
          "__name__": "haproxy_process_uptime_seconds",
          "agent_id": "e42be43b-286a-4c2a-a983-9b993df46084",
          "agent_type": "external-exporter",
          "cluster": "mariadb_lab_cluster",
          "environment": "lab",
          "node_name": "mariadb-db-01",
          "service_name": "haproxy-mariadb-01",
          "service_type": "external"
        },
        "value": [1794609096, "61"]
      },
      {
        "metric": {
          "__name__": "haproxy_process_uptime_seconds",
          "agent_id": "f6010804-d7fc-4e5f-9517-5a29d52cb59b",
          "agent_type": "external-exporter",
          "cluster": "mariadb_lab_cluster",
          "environment": "lab",
          "node_name": "mariadb-db-02",
          "service_name": "haproxy-mariadb-02",
          "service_type": "external"
        },
        "value": [1794609096, "54"]
      }
    ]
  }
}
```

---

## Lições

- **Métricas Nativas Dispensam Daemons de Terceiros:** Em versões antigas do HAProxy, era obrigatório rodar um binário separado (`haproxy_exporter`) escrito em Go para expor métricas ao Prometheus. O HAProxy moderno possui a funcionalidade embutida, economizando memória e eliminando pontos de falha.
- **Associe o External Exporter ao Cluster:** Ao rodar `pmm-admin add external`, passe sempre `--cluster=mariadb_lab_cluster`. Isso garante que o VictoriaMetrics do PMM inclua a etiqueta `cluster`, permitindo filtrar o HAProxy e o MariaDB no mesmo seletor de painel.
- **Métricas Mais Críticas do HAProxy:**
  - `haproxy_backend_status`: estado de cada nó (`UP = 1`, `DOWN = 0`).
  - `haproxy_frontend_connections_total`: taxa de novas conexões por segundo.
  - `haproxy_backend_current_queue`: conexões enfileiradas esperando liberação de backend.

---

## Pegadinhas

- **Falta de Reload do HAProxy:** Alterar `/etc/haproxy/haproxy.cfg` sem executar `systemctl reload haproxy` fará o endpoint `/metrics` retornar erro `503 Service Unavailable`, impedindo que o PMM colete as métricas.
- **Porta 8404 Fechada em Ambientes com Múltiplas Subredes:** O `pmm-agent` roda localmente no mesmo nó do HAProxy e coleta métricas via `127.0.0.1:8404`. Caso o HAProxy seja configurado com `bind 192.168.1.21:8404` em vez de `bind *:8404`, conexões por localhost falharão.
- **Conflito de Nomes com o Orquestrador do Postgres:** Lembre-se de nomear o serviço com prefixo claro (ex: `haproxy-mariadb-01`). No mesmo laboratório, existem orquestradores Patroni do PostgreSQL; nomes genéricos como `proxy-01` criam confusão nos seletores de métricas do Grafana.
