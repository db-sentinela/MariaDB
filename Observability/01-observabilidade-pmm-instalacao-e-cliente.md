# 01 — Instalação e Arquitetura do PMM Client no MariaDB

O **Percona Monitoring and Management (PMM 3)** é uma plataforma open-source de observabilidade de bancos de dados construída sob medida para ambientes de alta performance. Ele combina coleta de métricas de sistema operacional, telemetria detalhada de storage engines e análise granular de consultas (**Query Analytics - QAN**). Este documento detalha a arquitetura interna do PMM Client, o processo de instalação e a validação da conectividade segura com o servidor central.

---

## O que é

O ecossistema do PMM 3 é dividido em duas camadas principais:

### 1. PMM Server (Servidor Central)
Hospedado no nó de monitoramento (`192.168.1.16:8443`), ele consolida:
- **VictoriaMetrics:** Banco de dados de séries temporais (TSDB) de altíssima performance que armazena métricas de CPU, memória, I/O, status do MariaDB e variáveis de replicação.
- **ClickHouse:** Banco colunar analítico dedicado a armazenar os traços de execução e consultas capturados pelo QAN.
- **Grafana (v11+):** Interface visual unificada com dashboards especializados desenvolvidos pela Percona.

### 2. PMM Client (Nós Gerenciados)
Instalado em cada máquina do banco de dados (`mariadb-db-01`, `db-02`, `db-03`), é orquestrado pelo daemon `pmm-agent`. Ele gerencia os subagentes:
- **`vmagent`:** Coletor leve que envia métricas locais para o VictoriaMetrics central em modo *push* (porta segura 8443).
- **`node_exporter`:** Coleta métricas de hardware e SO (disco, rede, CPU, memória) rodando na porta local 42000 ou 42003.
- **`mysqld_exporter`:** Conecta ao MariaDB via credenciais de menor privilégio e extrai contadores de `SHOW GLOBAL STATUS`, variáveis de InnoDB, engine Aria e replicação Galera.
- **`mysql_perfschema_agent`:** Agente embutido responsável por ler o Performance Schema e enviar as queries para o Query Analytics.

---

## Por que importa

1. **Centralização sem Fragmentação:** Em vez de instalar e manter ferramentas isoladas (ex: Zabbix para OS, scripts manuais para slow log e dashboards genéricos), o PMM unifica telemetria de infraestrutura e aplicação em um único painel.
2. **Baixíssimo Overhead:** A arquitetura do PMM 3 utiliza o modelo *push* através do `vmagent`, minimizando o tráfego de rede e reduzindo o consumo de CPU nos nós de banco de dados para menos de 1%.
3. **Detecção Antecipada de Incidentes:** Permite correlacionar causas e efeitos instantaneamente — por exemplo, identificar se um aumento de latência na aplicação foi causado por um aumento de conexões no SO, por saturação de disco ou por uma query sem índice no MariaDB.

---

## Como reproduzir

Execute os passos a seguir no nó MariaDB para instalar e registrar o cliente no PMM Server:

### 1. Instalar o repositório Percona e o pacote PMM Client
No Ubuntu 26.04:
```bash
# Baixar o configurador oficial do repositorio Percona
wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
sudo dpkg -i percona-release_latest.generic_all.deb
sudo percona-release setup pmm3-client

# Atualizar indices e instalar o agente
sudo apt update
sudo apt install -y pmm-client
```

### 2. Registrar o nó no PMM Server
Conecte o agente local ao PMM Server (`192.168.1.16:8443`), informando as credenciais administrativas e permitindo o certificado TLS do laboratório:
```bash
sudo pmm-admin config \
  --server-insecure-tls \
  --server-url=https://admin:Admin#2026@192.168.1.16:8443 \
  192.168.1.21 generic mariadb-db-01
```

### 3. Verificar o status e a saúde dos agentes
```bash
sudo pmm-admin status
```

---

## O que você vê (saída real)

Ao rodar `pmm-admin status` no nó `mariadb-db-01`, o relatório consolida a saúde da conexão e a lista de agentes em execução:

```text
/usr/sbin/pmm-admin
Agent ID : 4dc4d1b6-bf21-44e3-a270-f13fbdc8da82
Node ID  : c3fb3838-6ba1-491d-88ec-a2b1b96928ff
Node name: mariadb-db-01

PMM Server:
	URL    : https://192.168.1.16:8443/
	Version: 3.10.0

PMM Client:
	Connected        : true
	Time drift       : -416.653µs
	Latency          : 1.422226ms
	Connection uptime: 100
	pmm-admin version: 3.10.0
	pmm-agent version: 3.10.0
Agents:
	0145353b-9157-4a73-bde5-77ac1daf1b92 nomad_agent                    Running        42000
	6a3e2bd6-92b3-4f60-b8c5-8eb85c5ac5fc mysqld_exporter                Running        42004
	8875f93b-87f3-47e2-a796-9375806fef16 mysql_perfschema_agent         Running        0
	9b40c7dd-634a-42b8-90f8-5549090ad6ab vmagent                        Running        42002
	b4c61a0a-15a8-43ac-abbf-a60e16a11616 node_exporter                  Running        42003
```

**Pontos de Atenção na Saída:**
- `Connected: true`: Comunicação gRPC/HTTPS estabelecida com sucesso.
- `Time drift: -416.653µs`: Diferença de relógio praticamente zero (essencial para que as séries temporais se alinhem).
- `Agents`: Todos os processos essenciais (`vmagent`, `node_exporter`, `mysqld_exporter`, `mysql_perfschema_agent`) em estado **Running**.

---

## Lições

- **Sincronização de Relógio (NTP):** O PMM depende criticamente de marcas de tempo alinhadas. Um *time drift* superior a algumas centenas de milissegundos gera anomalias gráficas e pode fazer métricas recentes serem descartadas pelo VictoriaMetrics. Garanta que o serviço `systemd-timesyncd` ou `chrony` esteja ativo em todos os nós.
- **Isolamento de Portas Locais:** Os exportadores (`mysqld_exporter`, `node_exporter`) escutam na interface local `127.0.0.1` em portas efêmeras na faixa 42000–42005. O tráfego de saída externo para o servidor ocorre estritamente na porta TCP 8443 (ou 443) em HTTPS.
- **Comando de Diagnóstico Rápido:** Para obter um pacote completo de logs para suporte ou análise interna, utilize `sudo pmm-admin summary`. Ele compila métricas, logs do agente e configurações mascaradas em um arquivo `.tar.gz`.

---

## Pegadinhas

- **O Certificado TLS Autoassinado:** O container Docker do PMM Server gera um certificado TLS autoassinado por padrão. Omitir a flag `--server-insecure-tls` durante o registro do cliente fará a conexão abortar com erro de certificado não confiável (`x509: certificate signed by unknown authority`).
- **Conflito de Nomes de Nó (`Node Name`):** O nome do nó informado no registro (`mariadb-db-01`) é a chave primária de identificação no Grafana. Se você clonar uma máquina virtual sem reconfigurar o PMM, o novo nó substituirá o registro do nó antigo, corrompendo a telemetria.
- **Portas Bloqueadas pelo Firewall:** Se houver regras de `iptables` ou `ufw` restritivas entre a rede das instâncias de banco e a VM do PMM Server, o `pmm-agent` falhará em conectar, gerando loops de reconexão no log (`connection refused` ou `context deadline exceeded`). A porta 8443 precisa estar explicitamente liberada na origem e no destino.
