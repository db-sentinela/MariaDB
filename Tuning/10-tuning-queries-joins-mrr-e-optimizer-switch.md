# 10 — Otimização de Junções (Joins), MRR e Flags do Optimizer Switch

Consultas relacionais envolvendo junções (`INNER JOIN`, `LEFT JOIN`) entre tabelas de grande porte representam uma das tarefas computacionalmente mais exigentes para um SGBD. No MariaDB 12, o otimizador baseado em custo (*Cost-Based Optimizer*) dispõe de algoritmos avançados de execução, como **Nested Loop Joins com cache**, **Multi-Range Read (MRR)** e dezenas de heurísticas configuráveis através da variável **`optimizer_switch`**.

---

## O que é

### 1. Algoritmos de Junção no MariaDB
Ao executar um join entre duas tabelas (ex: `clientes c JOIN pedidos p ON c.id = p.cliente_id`):
- **Tabela Externa (Driving Table):** O otimizador escolhe a tabela que possui a condição mais seletiva para iniciar a leitura.
- **Tabela Interna (Driven Table):** Para cada registro selecionado na driving table, o banco busca as linhas correspondentes na segunda tabela.
- Se a tabela interna possuir um índice adequado na chave de junção, o acesso é do tipo `ref` ou `eq_ref` (Nested Loop eficiente).
- Se a tabela interna não possuir índice, o MariaDB utiliza o **Block Nested Loop (BNL)** ou **Hash Join**, armazenando fatias da tabela externa em um buffer de memória (`join_buffer_size`) para comparar em lote.

### 2. Multi-Range Read (MRR)
Quando um índice secundário é varrido para recuperar linhas do índice primário (Clustered Index), os ponteiros de chave apontam para páginas aleatórias de dados. Em volumes grandes de dados, isso causa acessos desordenados de disco.
O algoritmo **MRR (`Multi-Range Read`)**:
1. Lê primeiro os blocos de chaves do índice secundário para um buffer de memória (`mrr_buffer_size`).
2. Ordena esses ponteiros sequencialmente pela ordem física de endereçamento em disco (RowID).
3. Executa as leituras na tabela primária de forma **estritamente linear e contínua**, transformando *random I/O* em *sequential I/O*.

### 3. A Diretiva `optimizer_switch`
É uma variável global e de sessão que controla detalhadamente quais recursos do otimizador estão habilitados:
- `mrr=on / mrr_cost_based=off`: Ativação do Multi-Range Read forçado.
- `index_condition_pushdown=on`: Empurramento de filtros para a engine.
- `join_cache_incremental=on` e `join_cache_hashed=on`: Junções otimizadas em memória com hashing.
- `extended_keys=on`: Aproveitamento da chave primária embutida em índices secundários.

---

## Por que importa

1. **Evita Inversões Trágicas de Join Order:** Se o otimizador escolher a tabela errada como *driving table* (ex: varrer 100.000 pedidos para depois filtrar 5 clientes, em vez de filtrar os 5 clientes primeiro e buscar apenas seus pedidos específicos), a consulta demorará minutos em vez de milissegundos.
2. **Transformação de I/O Aleatório em Sequencial:** Em discos SSD ou storages corporativos com alta latência de concorrência, o MRR acelera varreduras de intervalo em até 10 vezes.
3. **Controle Fino sem Reiniciar o Banco:** As flags de `optimizer_switch` e o tamanho de `join_buffer_size` podem ser ajustados dinamicamente em runtime para sessões analíticas ou globalmente para toda a instância.

---

## Como reproduzir

Execute os testes no nó `mariadb-db-01` usando as tabelas `clientes` (5.000 linhas) e `pedidos` (100.000 linhas):

### 1. Inspecionar as flags padrão do `optimizer_switch` no MariaDB 12
```sql
SELECT @@optimizer_switch\G
```

### 2. Avaliar o plano de execução de uma junção com índices apropriados
Crie os índices nas chaves de filtro e relacionamento:
```sql
USE lab_tuning;

CREATE INDEX idx_cli_cidade ON clientes(cidade);
CREATE INDEX idx_ped_status ON pedidos(status, cliente_id, valor);

-- Consulta unindo clientes de Curitiba com pedidos enviados
EXPLAIN SELECT c.nome, c.cidade, p.valor 
FROM clientes c 
JOIN pedidos p ON c.id = p.cliente_id 
WHERE c.cidade = 'Curitiba' AND p.status = 'ENVIADO'\G
```

### 3. Habilitar o Multi-Range Read (MRR) na sessão
```sql
SET optimizer_switch = 'mrr=on,mrr_cost_based=off';
```

### 4. Executar telemetria real com `ANALYZE FORMAT=JSON`
```sql
ANALYZE FORMAT=JSON SELECT c.nome, c.cidade, p.valor 
FROM clientes c 
JOIN pedidos p ON c.id = p.cliente_id 
WHERE c.cidade = 'Curitiba' AND p.status = 'ENVIADO'\G
```

---

## O que você vê (saída real)

### 1. Flags do `optimizer_switch` padrão do MariaDB 12.3:
```text
*************************** 1. row ***************************
@@optimizer_switch: index_merge=on,index_merge_union=on,index_merge_sort_union=on,
index_merge_intersection=on,index_merge_sort_intersection=off,index_condition_pushdown=on,
derived_merge=on,derived_with_keys=on,firstmatch=on,loosescan=on,duplicateweedout=on,
materialization=on,in_to_exists=on,semijoin=on,subquery_cache=on,mrr=off,
mrr_cost_based=off,outer_join_with_cache=on,join_cache_incremental=on,
join_cache_hashed=on,join_cache_bka=on,optimize_join_buffer_size=on,table_elimination=on,
extended_keys=on,rowid_filter=on
```

### 2. Plano de Execução do Join (`EXPLAIN`):
```text
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: c
         type: ALL
possible_keys: PRIMARY,idx_cli_cidade
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 5000
        Extra: Using where
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: p
         type: ref
possible_keys: idx_pedidos_status_data,idx_ped_cli_cod,idx_ped_cli_data,idx_ped_status
          key: idx_ped_status
      key_len: 86
          ref: const,lab_tuning.c.id
         rows: 20
        Extra: Using where; Using index
```
**Análise:** 
- O MariaDB escolheu `c` (clientes) como driving table.
- Para a tabela `p` (pedidos), usou o tipo de acesso ótimo `ref` através de `idx_ped_status` com `key_len: 86` (`status` + `cliente_id`).
- `Extra: Using where; Using index` confirma que a tabela interna foi resolvida como **Covering Index**, sem acessar a tabela física de pedidos.

### 3. Telemetria Real da Execução (`ANALYZE FORMAT=JSON`):
```json
ANALYZE: {
  "query_optimization": {
    "r_total_time_ms": 0.113621909
  },
  "query_block": {
    "select_id": 1,
    "cost": 4.8416512,
    "r_loops": 1,
    "r_total_time_ms": 2.248716694,
    "nested_loop": [
      {
        "table": {
          "table_name": "c",
          "access_type": "ALL",
          "loops": 1,
          "r_loops": 1,
          "rows": 5000,
          "r_rows": 5000,
          "cost": 0.8403864,
          "r_table_time_ms": 0.499269632,
          "r_engine_stats": {
            "pages_accessed": 21
          },
          "filtered": 20,
          "r_total_filtered": 20,
          "attached_condition": "c.cidade = 'Curitiba'",
          "r_filtered": 20
        }
      },
      {
        "table": {
          "table_name": "p",
          "access_type": "ref",
          "key": "idx_ped_status",
          "key_length": "86",
          "used_key_parts": ["status", "cliente_id"],
          "ref": ["const", "lab_tuning.c.id"],
          "loops": 1000,
          "r_loops": 1000,
          "rows": 20,
          "r_rows": 5,
          "cost": 4.0012648,
          "r_table_time_ms": 1.259982942,
          "r_engine_stats": {
            "pages_accessed": 2033
          },
          "filtered": 100,
          "r_total_filtered": 100,
          "attached_condition": "p.`status` = 'ENVIADO'",
          "r_filtered": 100,
          "using_index": true
        }
      }
    ]
  }
}
```
**Métricas Consolidadas:**
- A tabela `c` filtrou exatamente 1.000 clientes em Curitiba (`r_total_filtered: 20` sobre 5.000 linhas).
- A tabela `p` foi consultada 1.000 vezes (`r_loops: 1000`), encontrando uma média de 5 pedidos por cliente (`r_rows: 5`).
- O tempo total de execução foi de apenas **2,24 ms** para cruzar duas tabelas totalizando 105.000 linhas.

---

## Lições

- **Índice na Chave Estrangeira da Tabela Interna:** Em qualquer join, a tabela interna (do lado direito do loop) **deve** possuir um índice que inicie pela coluna usada no `ON` ou `WHERE`. Sem esse índice, a execução degrada para centenas de milhares de varreduras completas.
- **Forçar a Ordem de Junção com `STRAIGHT_JOIN`:** Em situações raras onde o otimizador escolher uma ordem ineficiente (devido a assimetrias de dados), você pode forçar o MariaDB a ler as tabelas estritamente na ordem em que foram escritas no SQL usando a cláusula `SELECT STRAIGHT_JOIN ...`.
- **Calibração de `join_buffer_size`:** Mantenha o valor padrão (256 KB a 1 MB). Aumentar `join_buffer_size` não melhora consultas que já usam índices (`ref`/`eq_ref`) e apenas consome RAM em conexões ociosas.

---

## Pegadinhas

- **`mrr=on` com `mrr_cost_based=on`:** O MRR vem desligado por padrão no MariaDB. Se você apenas habilitar `mrr=on`, o otimizador ainda usará a fórmula de custo que quase sempre decide não usar o MRR se os dados estiverem em cache de memória. Para forçar o MRR em leituras pesadas de disco, deve-se usar `SET optimizer_switch = 'mrr=on,mrr_cost_based=off'`.
- **Diferença de Charset e Collation nas Colunas de Join:** Se a coluna `clientes.id` for `INT` e a coluna `pedidos.cliente_id` for `BIGINT`, ou se colunas de texto tiverem collations diferentes (`utf8mb4_general_ci` vs `utf8mb4_unicode_ci`), o MariaDB **não poderá** usar o índice na junção, forçando um Block Nested Loop devastador.
- **Left Joins com Filtros na Tabela Direita no `WHERE`:** Colocar um filtro da tabela direita na cláusula `WHERE` (ex: `SELECT ... FROM c LEFT JOIN p ON c.id=p.cliente_id WHERE p.status='PAGO'`) converte silenciosamente o `LEFT JOIN` em um `INNER JOIN`, pois qualquer linha com `p.status IS NULL` será descartada. Coloque o filtro dentro da cláusula `ON` caso deseje preservar as linhas da tabela esquerda.
