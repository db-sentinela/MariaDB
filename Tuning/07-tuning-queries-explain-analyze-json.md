# 07 — Análise Profunda com EXPLAIN e ANALYZE FORMAT=JSON

Diagnosticar a performance de uma consulta SQL exige mais do que inspecionar o tempo total de resposta. O MariaDB disponibiliza duas ferramentas analíticas fundamentais: o **`EXPLAIN`** tradicional (que apresenta o plano estimado pelo otimizador de custos) e o moderno **`ANALYZE FORMAT=JSON`**, que executa a consulta de verdade no cluster e retorna a telemetria real de execução (linhas lidas, páginas acessadas e tempo gasto na engine).

---

## O que é

### 1. `EXPLAIN` (Estimativa Estática)
O `EXPLAIN` consulta as estatísticas do catálogo do InnoDB (`mysql.innodb_table_stats` e `mysql.innodb_index_stats`) para estimar o caminho de menor custo computacional (`cost`). Ele informa:
- `type`: O tipo de acesso à tabela (ex: `ALL` para varredura completa, `index` para varredura do índice, `range` para intervalos, `ref` para buscas por igualdade, `eq_ref` para chaves únicas).
- `possible_keys` e `key`: Índices avaliados vs o índice efetivamente escolhido.
- `rows`: Estimativa estatística de linhas a serem examinadas.
- `Extra`: Indicadores de operações adicionais (`Using where`, `Using index`, `Using filesort`, `Using temporary`).

### 2. `ANALYZE FORMAT=JSON` (Telemetria Real de Execução)
Enquanto o `EXPLAIN` apenas projeta o que *deveria* acontecer, o `ANALYZE FORMAT=JSON` executa a query e injeta contadores de tempo real na árvore JSON do plano:
- `r_total_time_ms`: Tempo total gasto pelo nó na execução.
- `r_table_time_ms` e `r_other_time_ms`: Tempo isolado gasto dentro da storage engine (InnoDB) versus tempo de processamento das expressões SQL no servidor.
- `r_rows`: O número **real** de linhas lidas da engine (que frequentemente diverge das estimativas de `rows`).
- `r_filtered`: O percentual **real** de linhas que atenderam à condição de filtro (`WHERE`).
- `pages_accessed`: Quantidade física de páginas de 16 KB acessadas no Buffer Pool do InnoDB.

---

## Por que importa

1. **Estatísticas Desatualizadas Enganam o Otimizador:** Em bancos transacionais de alta rotatividade, as estatísticas estatísticas de `rows` podem indicar 5.000 linhas quando na realidade a query processará 500.000 linhas. O `ANALYZE` expõe a discrepância imediatamente (`rows` vs `r_rows`).
2. **Identificação Precisa do Gargalo (Engine vs Server):** Se `r_table_time_ms` representar 90% do tempo, o gargalo é I/O e varredura de páginas no InnoDB (solução: índices ou buffer pool). Se `r_other_time_ms` for alto, o gargalo é CPU no servidor MariaDB (solução: simplificação de expressões, joins ou ordenação).
3. **Métrica `pages_accessed`:** Permite quantificar a pressão exercida no Buffer Pool. Uma query eficiente toca dezenas de páginas; uma query mal otimizada consome centenas de milhares de páginas, causando *cache eviction*.

---

## Como reproduzir

Execute os testes no nó `mariadb-db-01` usando a tabela `pedidos` (100.000 linhas):

### 1. Preparar a tabela de teste sem índices secundários
```sql
CREATE DATABASE IF NOT EXISTS lab_tuning;
USE lab_tuning;

CREATE OR REPLACE TABLE pedidos (
    id INT PRIMARY KEY AUTO_INCREMENT,
    cliente_id INT NOT NULL,
    status VARCHAR(20) NOT NULL,
    valor DECIMAL(10,2) NOT NULL,
    criado_em DATETIME NOT NULL,
    codigo_rastreio VARCHAR(50) NULL
) ENGINE=InnoDB;

-- Popula 100.000 registros
INSERT INTO pedidos (cliente_id, status, valor, criado_em, codigo_rastreio)
SELECT 
    1 + (seq % 5000),
    ELT(1 + (seq % 4), 'PENDENTE', 'PAGO', 'CANCELADO', 'ENVIADO'),
    ROUND(10 + (RAND() * 990), 2),
    DATE_SUB(NOW(), INTERVAL (seq % 365) DAY),
    IF(seq % 4 = 0, CONCAT('BR', LPAD(seq, 9, '0'), 'XP'), NULL)
FROM seq_1_to_100000;
```

### 2. Executar `EXPLAIN` tabular da query lenta
```sql
EXPLAIN SELECT status, SUM(valor) 
FROM pedidos 
WHERE criado_em >= '2026-06-01' AND status = 'PAGO' 
GROUP BY status\G
```

### 3. Executar `ANALYZE FORMAT=JSON` para inspecionar métricas reais
```sql
ANALYZE FORMAT=JSON SELECT status, SUM(valor) 
FROM pedidos 
WHERE criado_em >= '2026-06-01' AND status = 'PAGO' 
GROUP BY status\G
```

### 4. Criar índice composto e reavaliar métricas reais
```sql
-- Indice cobrindo as colunas do filtro e agregacao
CREATE INDEX idx_pedidos_status_data ON pedidos(status, criado_em, valor);

-- Nova inspecao com ANALYZE
ANALYZE FORMAT=JSON SELECT status, SUM(valor) 
FROM pedidos 
WHERE criado_em >= '2026-06-01' AND status = 'PAGO' 
GROUP BY status\G
```

---

## O que você vê (saída real)

### 1. `EXPLAIN` Tabular Tradicional (Sem Índice):
```text
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: pedidos
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 100116
        Extra: Using where
```
*(O plano apenas aponta `type: ALL` e uma estimativa de 100.116 linhas, sem detalhar tempo ou páginas).*

### 2. `ANALYZE FORMAT=JSON` (Sem Índice — Execução Real):
```json
ANALYZE: {
  "query_optimization": {
    "r_total_time_ms": 0.054137261
  },
  "query_block": {
    "select_id": 1,
    "cost": 16.6306524,
    "r_loops": 1,
    "r_total_time_ms": 12.19223193,
    "nested_loop": [
      {
        "table": {
          "table_name": "pedidos",
          "access_type": "ALL",
          "loops": 1,
          "r_loops": 1,
          "rows": 100116,
          "r_rows": 100000,
          "cost": 16.6306524,
          "r_table_time_ms": 8.946770453,
          "r_other_time_ms": 3.240254809,
          "r_engine_stats": {
            "pages_accessed": 325
          },
          "filtered": 100,
          "r_total_filtered": 11.372,
          "attached_condition": "pedidos.criado_em >= '2026-06-01' and pedidos.`status` = 'PAGO'",
          "r_filtered": 11.372
        }
      }
    ]
  }
}
```
**Análise:** O banco examinou `r_rows: 100000`, gastou `8.94 ms` lendo `325 páginas` do disco/buffer e filtrou apenas `11.37%` dos dados (`r_filtered: 11.372`). Quase 89.000 linhas foram lidas para serem descartadas.

### 3. `ANALYZE FORMAT=JSON` (Com Índice Otimizado):
```json
ANALYZE: {
  "query_optimization": {
    "r_total_time_ms": 0.099137729
  },
  "query_block": {
    "select_id": 1,
    "cost": 3.48805577,
    "r_loops": 1,
    "r_total_time_ms": 2.923653645,
    "nested_loop": [
      {
        "table": {
          "table_name": "pedidos",
          "access_type": "range",
          "possible_keys": ["idx_pedidos_status_data"],
          "key": "idx_pedidos_status_data",
          "key_length": "87",
          "used_key_parts": ["status", "criado_em"],
          "loops": 1,
          "r_loops": 1,
          "rows": 23378,
          "r_rows": 11372,
          "cost": 3.48805577,
          "r_table_time_ms": 1.013520697,
          "r_other_time_ms": 1.90470515,
          "r_engine_stats": {
            "pages_accessed": 31
          },
          "filtered": 100,
          "r_total_filtered": 100,
          "attached_condition": "pedidos.criado_em >= '2026-06-01' and pedidos.`status` = 'PAGO'",
          "r_filtered": 100,
          "using_index": true
        }
      }
    ]
  }
}
```
**Ganhos Reais Homologados:**
- Custo computacional caiu de **16.63** para **3.48** (redução de 79%).
- Tempo na engine (`r_table_time_ms`) caiu de **8.94 ms** para **1.01 ms** (quase 9x mais rápido).
- Páginas acessadas caíram de **325** para **31** (alívio de 90% na pressão de memória).
- `using_index: true` confirma que a leitura ocorreu inteiramente na árvore do índice (Covering Index).

---

## Lições

- **`ANALYZE` Executa a Query:** Lembre-se sempre de que `ANALYZE` **roda a instrução de verdade** contra os dados. Se você rodar `ANALYZE DELETE FROM ...` ou `ANALYZE UPDATE ...`, as linhas **serão modificadas**. Em comandos DML, execute dentro de uma transação com `ROLLBACK` ou use cópia de homologação.
- **Divergência de `rows` vs `r_rows`:** Se o valor estimado em `rows` for dezenas de vezes diferente de `r_rows`, execute `ANALYZE TABLE <tabela>` para atualizar as estatísticas do InnoDB e evitar escolhas catastróficas de planos de execução pelo otimizador.
- **Relação `r_engine_stats.pages_accessed`:** Monitore o total de páginas lidas. Em servidores de produção com alta taxa de concorrência, o número de páginas tocadas por query é o indicador número 1 de saturação do Buffer Pool.

---

## Pegadinhas

- **`ANALYZE` em Queries Lentas Trava a Sessão:** Como o `ANALYZE` executa a query até o fim, rodá-lo em uma consulta não indexada em uma tabela de 50 milhões de linhas congelará o terminal por horas. Comece sempre com `EXPLAIN` estático e só passe para `ANALYZE` quando entender o impacto do plano.
- **A Pegadinha do Cache de Query / Buffer Pool Warm:** A primeira execução de um `ANALYZE` pode ter `r_total_time_ms` mais alto devido à busca inicial no subsistema de armazenamento. Repita o comando 2 ou 3 vezes para obter a média em estado estável de memória.
- **Formato JSON no MariaDB vs MySQL:** A estrutura da árvore JSON do MariaDB possui nós específicos como `r_engine_stats` e `r_other_time_ms` que não existem no MySQL da Oracle. Use ferramentas e parsers compatíveis com o dialeto MariaDB.
