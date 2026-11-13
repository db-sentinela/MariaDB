# 02 — Monitoramento de Instância Single (Standalone) no PMM

Monitorar uma instância isolada (standalone) do MariaDB exige duas configurações fundamentais: a criação de um usuário com **Princípio de Menor Privilégio** para os exportadores e a correta instrumentação do banco para alimentar o **Query Analytics (QAN)**. Este documento apresenta o procedimento operacional homologado para adicionar serviços MySQL/MariaDB ao PMM 3 garantindo segurança e telemetria de consultas.

---

## O que é

Para coletar métricas internas do motor InnoDB, variáveis de status e dados de consultas, o PMM Client precisa de uma conta de acesso ao banco. 
- **Nunca utilize a conta `root`:** Executar daemons de monitoramento como superusuário viola os padrões de conformidade (PCI-DSS, ISO 27001) e expõe a infraestrutura caso o processo seja comprometido.
- **Usuário Dedicado `pmm`:** Criado com privilégios estritamente necessários para ler metadados, consultar tabelas do sistema (`information_schema`, `performance_schema`) e monitorar réplicas.
- **Query Analytics (QAN) via Performance Schema:** O MariaDB 12 possui o subsistema `performance_schema` nativo e eficiente. Configurar o PMM para ler as consultas diretamente da tabela `events_statements_summary_by_digest` elimina a necessidade de habilitar o pesado log textual de queries lentas (*Slow Query Log*).

---

## Por que importa

1. **Segurança de Acesso e Contenção de Conexões:** O usuário `pmm` é configurado com a cláusula `WITH MAX_USER_CONNECTIONS 10`. Isso garante que, mesmo em caso de falha de script ou loop no agente, o monitoramento nunca consumirá as conexões disponíveis do banco, preservando o pool da aplicação.
2. **QAN de Baixo Impacto Computacional:** Habilitar `--query-source=perfschema` permite que o PMM capture a taxa de queries, latência média, percentis e planos de execução em memória, mantendo o consumo de disco zerado.
3. **Métricas Detalhadas de Tabelas:** O `mysqld_exporter` coleta estatísticas de tamanho, fragmentação e I/O por tabela, permitindo identificar tabelas que necessitam de desfragmentação (`OPTIMIZE TABLE`) ou particionamento.

---

## Como reproduzir

Execute os passos a seguir no servidor MariaDB para criar o usuário e registrar o serviço standalone no PMM:

### 1. Criar o usuário de monitoramento no MariaDB com menor privilégio
Conecte-se ao MariaDB via terminal administrativo local (`sudo mariadb`) e execute:

```sql
-- Cria o usuario pmm restrito
CREATE USER IF NOT EXISTS 'pmm'@'127.0.0.1' IDENTIFIED BY 'Pmm#2026' WITH MAX_USER_CONNECTIONS 10;
CREATE USER IF NOT EXISTS 'pmm'@'localhost' IDENTIFIED BY 'Pmm#2026' WITH MAX_USER_CONNECTIONS 10;

-- Concede apenas os privilegios operacionais estritamente necessarios
GRANT SELECT, PROCESS, REPLICATION CLIENT, RELOAD, BINLOG MONITOR ON *.* TO 'pmm'@'127.0.0.1';
GRANT SELECT, PROCESS, REPLICATION CLIENT, RELOAD, BINLOG MONITOR ON *.* TO 'pmm'@'localhost';

-- Permite leitura das tabelas de telemetria do Performance Schema
GRANT SELECT ON performance_schema.* TO 'pmm'@'127.0.0.1';
GRANT SELECT ON performance_schema.* TO 'pmm'@'localhost';

FLUSH PRIVILEGES;
```

### 2. Garantir que os instrumentos do Performance Schema estão ativos
Certifique-se de que o arquivo `/etc/mysql/mariadb.conf.d/60-pmm.cnf` possui:

```ini
[mysqld]
performance_schema = ON
performance-schema-instrument = 'statement/%=ON'
performance-schema-consumer-statements-digest = ON
performance-schema-consumer-events-statements-current = ON
performance-schema-consumer-events-statements-history = ON
```

### 3. Adicionar o serviço no PMM Client
Registre a instância informando o nome do serviço, o usuário `pmm`, a senha e o modo de QAN:

```bash
sudo pmm-admin add mysql \
  --service-name=mariadb-standalone-01 \
  --host=127.0.0.1 \
  --port=3306 \
  --username=pmm \
  --password='Pmm#2026' \
  --query-source=perfschema \
  --environment=production
```

### 4. Validar o funcionamento dos agentes
```bash
sudo pmm-admin list
```

---

## O que você vê (saída real)

### Confirmação da adição do serviço no PMM Client:
```text
MySQL Service added.
Service ID  : 8814a8f7-3458-41d9-84c6-de8dc06343ff
Service name: mariadb-standalone-01

Table statistics collection enabled (the limit is 1000, the actual table count is 312).
```

### Lista de serviços e agentes confirmada via `pmm-admin list`:
```text
Service type        Service name             Address and port         Service ID
MySQL               mariadb-standalone-01    127.0.0.1:3306           8814a8f7-3458-41d9-84c6-de8dc06343ff

Agent type                    Status           Metrics Mode        Agent ID                                    Service ID                                  Port
pmm_agent                     Connected                            4dc4d1b6-bf21-44e3-a270-f13fbdc8da82                                                    0 
node_exporter                 Running          push                b4c61a0a-15a8-43ac-abbf-a60e16a11616                                                    42003 
mysqld_exporter               Running          push                6a3e2bd6-92b3-4f60-b8c5-8eb85c5ac5fc        8814a8f7-3458-41d9-84c6-de8dc06343ff        42004 
mysql_perfschema_agent        Running                              8875f93b-87f3-47e2-a796-9375806fef16        8814a8f7-3458-41d9-84c6-de8dc06343ff        0 
vmagent                       Running          push                9b40c7dd-634a-42b8-90f8-5549090ad6ab                                                    42002 
nomad_agent                   Running                              0145353b-9157-4a73-bde5-77ac1daf1b92                                                    42000
```
*(Ambos `mysqld_exporter` e `mysql_perfschema_agent` em estado **Running** com portas atribuídas).*

---

## Lições

- **Limite de Coleta de Tabelas:** O PMM possui um limite de segurança (1.000 tabelas por padrão) para evitar que instâncias com dezenas de milhares de tabelas saturem o banco ao consultar `information_schema.TABLES`. Em ambientes com muitas tabelas, o PMM desativa automaticamente a coleta individual de métricas de tabelas para preservar a estabilidade.
- **Vantagem de `perfschema` sobre `slowlog`:** O modo `slowlog` precisa que o MariaDB grave logs em disco e o agente leia continuamente o arquivo de texto. Em bancos com pico de escrita, isso concorre por I/O. O `perfschema` consome buffers de memória pré-alocados de forma extremamente eficiente.
- **Permissão `BINLOG MONITOR` no MariaDB 12:** Em versões modernas do MariaDB, a permissão `REPLICATION CLIENT` foi subdividida. A permissão `BINLOG MONITOR` é obrigatória para que o exporter possa auditar o tamanho e as coordenadas do binlog sem necessitar de privilégios `SUPER`.

---

## Pegadinhas

- **O Agente em Estado `Waiting`:** Se o `mysql_perfschema_agent` aparecer em estado `Waiting` em vez de `Running`, verifique:
  1. Se os consumidores de digest estão ativos no `performance_schema`.
  2. Se a senha do usuário `pmm` foi alterada no banco mas não atualizada no PMM.
  3. Se o banco está recusando conexões locais na porta indicada.
- **Uso de Senhas com Caracteres Especiais no Shell:** Se a senha do banco contiver caracteres como `#`, `$` ou `!`, passe o valor sempre entre **aspas simples** no comando bash (`--password='Pmm#2026'`). Omitir as aspas fará o shell interpretar o `#` como comentário ou o `$` como variável, corrompendo a senha enviada ao agente.
- **Falta de `FLUSH PRIVILEGES`:** Em algumas versões do MariaDB, após criar usuários com `MAX_USER_CONNECTIONS`, se `FLUSH PRIVILEGES` não for executado, conexões simultâneas do exporter podem ser rejeitadas por falha de sincronização do cache de privilégios.
