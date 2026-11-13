# 05 — Plugins de Tuning: Query Response Time (QRT) e Metadata Lock Info

Fazer o tuning de um banco de dados "no escuro", sem dados precisos de telemetria, é receita para decisões equivocadas. Embora o **Slow Query Log** tradicional seja muito utilizado, ele possui deficiências graves: só registra consultas que ultrapassam um limiar estático (ex: > 1s) e introduz overhead pesado de escrita em disco. Este documento aborda dois plugins nativos de observabilidade e diagnóstico de ponta no MariaDB: o **`query_response_time` (QRT)**, que calcula histogramas logarítmicos de latência em memória, e o **`metadata_lock_info`**, que expõe a fila oculta de contenção de DDL e DML.

---

## O que é

### 1. Plugin Query Response Time (QRT)
Originalmente desenvolvido pela Percona e portado de forma nativa para o MariaDB, o plugin `query_response_time` monitora o tempo de execução de **todas** as queries processadas pelo servidor, agrupando-as em faixas de tempo (*buckets* logarítmicos) em memória. 
Ele disponibiliza as visões:
- `information_schema.QUERY_RESPONSE_TIME`: distribuição geral de latência.
- `information_schema.QUERY_RESPONSE_TIME_READ`: latência exclusiva de comandos de leitura (`SELECT`).
- `information_schema.QUERY_RESPONSE_TIME_WRITE`: latência exclusiva de comandos de escrita (`INSERT`, `UPDATE`, `DELETE`).

A base da escala logarítmica é definida por `query_response_time_range_base` (padrão 10), gerando baldes que vão de frações de microssegundos ($10^{-6}$ s) até segundos.

### 2. Plugin Metadata Lock Info
O Metadata Lock (MDL) protege a consistência estrutural das tabelas. Quando uma transação mantém um `SELECT` aberto e uma migração tenta rodar `ALTER TABLE`, todas as consultas subsequentes na tabela enfileiram em estado `Waiting for table metadata lock`. O plugin `metadata_lock_info` expõe a tabela `information_schema.METADATA_LOCK_INFO`, permitindo correlacionar imediatamente a thread bloqueadora, o tipo de lock e a tabela afetada.

---

## Por que importa

1. **Visibilidade Real de SLA / SLO (Percentis p50, p95, p99):** O Slow Log só mostra anomalias extremas. Se uma aplicação web possui um SLO de responder em até 10 ms e 90% das queries estão demorando 800 ms, o Slow Log com `long_query_time=1` reportará **zero** ocorrências. O QRT expõe a degradação sistêmica instantaneamente.
2. **Custo Computacional Quase Nulo:** Como os contadores do QRT são atualizados em tabelas internas de memória sem gerar I/O de disco, ele pode permanecer ativo continuamente em produção sem afetar o desempenho da instância.
3. **Destravamento Rápido de DDLs em Janelas Críticas:** Em vez de adivinhar qual conexão está segurando um lock de metadados durante uma manutenção, o plugin `metadata_lock_info` aponta a thread exata para a aplicação de `KILL` imediato.

---

## Como reproduzir

Execute os passos a seguir no nó `mariadb-db-01` para instalar, habilitar e coletar dados com os plugins:

### 1. Instalar os plugins em tempo de execução
```sql
-- Instala o subsistema de Query Response Time
INSTALL SONAME 'query_response_time';

-- Instala o visualizador de locks de metadados
INSTALL SONAME 'metadata_lock_info';
```

Verifique se os plugins foram carregados com status `ACTIVE`:
```sql
SHOW PLUGINS WHERE Name IN ('QUERY_RESPONSE_TIME', 'METADATA_LOCK_INFO');
```

### 2. Habilitar a coleta estatística do QRT
Por padrão de segurança, o plugin é instalado com a coleta desligada. É necessário ativá-lo:
```sql
-- Ativa a contagem
SET GLOBAL query_response_time_stats = ON;

-- Inspecionar configuracoes de base e persistencia
SELECT @@query_response_time_stats, @@query_response_time_range_base;
```

### 3. Gerar consultas com diferentes faixas de latência
Execute comandos variados para preencher os diferentes *buckets* do histograma:
```sql
CREATE DATABASE IF NOT EXISTS lab_tuning;
USE lab_tuning;

-- Query ultra-rapida (< 1ms)
SELECT 1;

-- Query com latencia de ~5ms
SELECT SLEEP(0.005);

-- Query com latencia de ~20ms
SELECT SLEEP(0.02);

-- Query com latencia de ~100ms
SELECT SLEEP(0.1);
```

### 4. Consultar o histograma de latência consolidado
```sql
SELECT 
    TIME, 
    COUNT, 
    ROUND(TOTAL, 6) AS TOTAL_TIME_SEC 
FROM information_schema.QUERY_RESPONSE_TIME 
WHERE count > 0;
```

### 5. Inspecionar a fila de MDL em tempo real
```sql
SELECT * FROM information_schema.METADATA_LOCK_INFO;
```

### 6. Como persistir a ativação dos plugins no MariaDB
Para garantir que os plugins permaneçam carregados e ativos após reinicializações, adicione ao `/etc/mysql/mariadb.conf.d/80-tuning.cnf`:

```ini
[mariadb]
# Carrega automaticamente os plugins na inicializacao
plugin_load_add = query_response_time
plugin_load_add = metadata_lock_info

# Ativa a coleta do QRT no boot
query_response_time_stats = ON
query_response_time_range_base = 10
```

---

## O que você vê (saída real)

### Confirmação da instalação dos plugins:
```text
+----------------------+--------+--------------------+------------------------+---------+
| Name                 | Status | Type               | Library                | License |
+----------------------+--------+--------------------+------------------------+---------+
| QUERY_RESPONSE_TIME  | ACTIVE | INFORMATION SCHEMA | query_response_time.so | GPL     |
| METADATA_LOCK_INFO   | ACTIVE | INFORMATION SCHEMA | metadata_lock_info.so  | GPL     |
+----------------------+--------+--------------------+------------------------+---------+
```

### Ativação da coleta estatística:
```sql
mariadb -e "SET GLOBAL query_response_time_stats = ON;"
mariadb -e "SELECT @@query_response_time_stats, @@query_response_time_range_base;"
```
Saída real:
```text
+----------------------------+--------------------------------+
| @@query_response_time_stats| @@query_response_time_range_base|
+----------------------------+--------------------------------+
|                          1 |                             10 |
+----------------------------+--------------------------------+
```

### Histograma real de latência extraído de `information_schema.QUERY_RESPONSE_TIME`:
```sql
SELECT * FROM information_schema.QUERY_RESPONSE_TIME WHERE count > 0;
```
Saída real capturada no laboratório:
```text
+----------------+-------+----------------+
| TIME           | COUNT | TOTAL          |
+----------------+-------+----------------+
|       0.000001 |     1 |       0.000000 |
|       0.000010 |     3 |       0.000009 |
|       0.000100 |     3 |       0.000190 |
|       0.001000 |     1 |       0.000325 |
|       0.010000 |     1 |       0.005140 |
|       0.100000 |     1 |       0.046104 |
|       1.000000 |     1 |       0.125380 |
+----------------+-------+----------------+
```

**Como interpretar os dados:**
- Na linha `0.010000`: Houve 1 consulta executada no intervalo entre 1 ms ($0.001$ s) e 10 ms ($0.010$ s), somando um tempo total acumulado de $0.005140$ s (5.14 ms).
- Na linha `1.000000`: Houve 1 consulta executada no intervalo entre 100 ms ($0.100$ s) e 1 segundo ($1.000$ s), consumindo $0.125380$ s (125.38 ms).

---

## Lições

- **Diagnóstico Não Intrusivo de Performance:** O QRT permite que a equipe de infraestrutura analise a experiência real do usuário em percentis sem impactar o disco com gigabytes de logs de texto.
- **Zerar Contadores para Benchmarks:** Os contadores do QRT são cumulativos desde o momento da ativação. Antes de iniciar um teste de carga ou validação de nova versão de software, zere o histograma com o comando:
  ```sql
  SET GLOBAL query_response_time_flush = ON;
  ```
- **Persistência em Tabela de Sistema:** Uma vez executado `INSTALL SONAME 'query_response_time'`, o MariaDB registra o plugin na tabela `mysql.plugin`. Porém, a variável de runtime `query_response_time_stats = ON` voltará como `OFF` no reboot caso não seja adicionada explicitamente ao arquivo `.cnf`.

---

## Pegadinhas

- **O Plugin Vem Desligado por Padrão:** A armadilha mais frequente com o `query_response_time` é instalar o plugin via `INSTALL SONAME` e esperar que as métricas apareçam. Se você esquecer de rodar `SET GLOBAL query_response_time_stats = ON`, a tabela `information_schema.QUERY_RESPONSE_TIME` permanecerá completamente vazia.
- **Conflito de Nomes em Bibliotecas:** Nunca use `INSTALL PLUGIN query_response_time SONAME 'query_response_time.so'`. Como a biblioteca contém múltiplos componentes internos (READ, WRITE, AUDIT), a sintaxe correta e segura é sempre `INSTALL SONAME 'query_response_time'`, que carrega o bundle completo de uma só vez.
- **Atraso de Resolução do Metadata Lock:** O plugin `metadata_lock_info` mostra locks ativos em nível de SQL. No entanto, no Galera Cluster, DDLs em tabelas replicadas operam via **TOI (Total Order Isolation)** e realizam *BF-abort* (aborto por força bruta) da transação concorrente em vez de aguardar o lock indefinidamente. O `metadata_lock_info` é especialmente útil em tabelas locais ou nós em manutenção.
