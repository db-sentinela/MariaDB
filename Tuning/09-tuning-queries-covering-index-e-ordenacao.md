# 09 — Índices de Cobertura (Covering Index), Filesort e Paginação Eficiente

O objetivo supremo do tuning de consultas no InnoDB é evitar o acesso à tabela física (*Clustered Index Lookup*). Quando um índice contém todas as colunas requisitadas por uma consulta, o MariaDB atinge o estado de **Covering Index** (`Using index`), resolvendo a query inteiramente na memória da árvore B-Tree. Além disso, estruturar índices para atender à cláusula `ORDER BY` elimina operações pesadas de ordenação em memória ou disco (**`Using filesort`**).

---

## O que é

### 1. Covering Index (Índice de Cobertura)
Em tabelas InnoDB, a tabela *é* o índice primário (Clustered Index). Os índices secundários armazenam as colunas indexadas mais o valor da Chave Primária (`PRIMARY KEY`).
- Quando uma consulta seleciona colunas que **não** estão no índice secundário, o InnoDB precisa usar a chave primária para buscar a linha completa no clustered index (operação chamada de *Bookmark Lookup* ou *Secondary-to-Primary Lookup*).
- Quando todas as colunas do `SELECT`, `WHERE`, `JOIN` e `ORDER BY` estão presentes no próprio índice secundário (ou na chave primária), o InnoDB não precisa tocar na tabela. O plano de execução exibe **`Extra: Using index`**.

### 2. Eliminação de `Using filesort`
Quando o otimizador precisa ordenar o resultado e os dados lidos não estão na ordem solicitada:
1. Ele aloca um bloco de memória chamado `sort_buffer_size`.
2. Coleta os registros e executa um algoritmo de ordenação (QuickSort).
3. Se os dados ultrapassarem o `sort_buffer_size`, ele cria arquivos temporários em disco.
Esse processo é reportado como **`Using filesort`**. Quando o índice já está estruturado na ordem requerida pelo `ORDER BY`, o MariaDB lê os dados diretamente na sequência da B-Tree, eliminando o filesort.

### 3. Keyset Pagination vs OFFSET/LIMIT Tradicional
O padrão `LIMIT 90000, 20` força o MariaDB a varrer 90.020 registros na B-Tree, carregar todas as páginas no Buffer Pool e descartar as primeiras 90.000 para retornar apenas 20.
O padrão **Keyset Pagination** (ou busca por cursor / *seek method*) utiliza o último identificador lido (`WHERE id > 90000 LIMIT 20`), navegando diretamente na chave da árvore B-Tree e examinando exatamente 20 registros.

---

## Por que importa

1. **Ganho de até 450x em Paginação de Grandes Volumes:** Conforme demonstrado nos testes de laboratório, paginar a página 4.500 de uma tabela com `LIMIT 90000, 20` demorou **9,00 ms** lendo 90.020 linhas e 292 páginas de buffer. Com Keyset Pagination, a mesma consulta executou em **0,02 ms** lendo apenas 20 linhas e 26 páginas.
2. **Zero I/O no Bookmark Lookup:** Em índices de cobertura, o throughput de leitura multiplica por 3x a 5x porque as páginas de índice secundário são muito menores e mais densas que as páginas de dados completas.
3. **Economia de CPU com Eliminação de Filesort:** Ordenações em lote com `Using filesort` consomem ciclos massivos de processador e podem estourar a memória se dezenas de conexões concorrentes requisitarem ordenações simultâneas.

---

## Como reproduzir

Execute os testes no nó `mariadb-db-01` usando a tabela `pedidos`:

### 1. Validar um Covering Index puro (`Using index`)
Execute uma consulta selecionando apenas a chave primária e as colunas do índice secundário `idx_ped_cli_cod (cliente_id, codigo_rastreio)`:
```sql
USE lab_tuning;
EXPLAIN SELECT id, cliente_id, codigo_rastreio 
FROM pedidos 
WHERE cliente_id = 500\G
```
*(Observe `Extra: Using index` — a chave primária `id` é coberta automaticamente pelo InnoDB).*

### 2. Demonstrar o impacto de `Using filesort`
Execute uma busca filtrando por cliente e ordenando pela data de criação (coluna não indexada em conjunto):
```sql
EXPLAIN SELECT id, cliente_id, valor, criado_em 
FROM pedidos 
WHERE cliente_id = 500 
ORDER BY criado_em DESC\G
```
*(Observe a presença de `Extra: Using where; Using filesort`).*

### 3. Eliminar o Filesort criando índice composto ordenado
Crie o índice que atende simultaneamente ao filtro e à ordenação:
```sql
CREATE INDEX idx_ped_cli_data ON pedidos(cliente_id, criado_em);

-- Repita o EXPLAIN
EXPLAIN SELECT id, cliente_id, valor, criado_em 
FROM pedidos 
WHERE cliente_id = 500 
ORDER BY criado_em DESC\G
```
*(Observe que `Using filesort` desapareceu completamente).*

### 4. Benchmark: Paginação com `OFFSET` vs `Keyset Pagination`
Execute o `ANALYZE FORMAT=JSON` comparando a paginação profunda tradicional com a abordagem por cursor:

```sql
-- 4A: Paginacao Tradicional com OFFSET profundo
ANALYZE FORMAT=JSON SELECT id, valor FROM pedidos ORDER BY id LIMIT 90000, 20\G

-- 4B: Keyset Pagination (Seek Method)
ANALYZE FORMAT=JSON SELECT id, valor FROM pedidos WHERE id > 90000 ORDER BY id LIMIT 20\G
```

---

## O que você vê (saída real)

### 1. Covering Index (`Extra: Using index`):
```text
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: pedidos
         type: ref
possible_keys: idx_ped_cli_cod
          key: idx_ped_cli_cod
      key_len: 4
          ref: const
         rows: 20
        Extra: Using index
```
**Análise:** O MariaDB leu apenas o índice secundário. A tabela primária não foi tocada.

### 2. Consulta sofrendo `Using filesort`:
```text
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: pedidos
         type: ref
possible_keys: idx_ped_cli_cod
          key: idx_ped_cli_cod
      key_len: 4
          ref: const
         rows: 20
        Extra: Using where; Using filesort
```

### 3. Filesort eliminado com `idx_ped_cli_data`:
```text
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: pedidos
         type: ref
possible_keys: idx_ped_cli_cod,idx_ped_cli_data
          key: idx_ped_cli_data
      key_len: 4
          ref: const
         rows: 20
        Extra: Using where
```
**Análise:** O MariaDB aproveitou a ordem natural das folhas da B-Tree, entregando os registros já ordenados sem alocar `sort_buffer`.

### 4. Comparativo Real de Paginação (ANALYZE JSON):

#### Paginação Tradicional com `LIMIT 90000, 20`:
```json
{
  "select_id": 1,
  "cost": 14.94261778,
  "r_loops": 1,
  "r_total_time_ms": 9.005018769,
  "table": {
    "table_name": "pedidos",
    "access_type": "index",
    "rows": 100116,
    "r_rows": 90020,
    "r_engine_stats": {
      "pages_accessed": 292
    }
  }
}
```

#### Keyset Pagination com `WHERE id > 90000 LIMIT 20`:
```json
{
  "select_id": 1,
  "cost": 14.94261778,
  "r_loops": 1,
  "r_total_time_ms": 0.020143162,
  "table": {
    "table_name": "pedidos",
    "access_type": "range",
    "rows": 100116,
    "r_rows": 20,
    "r_engine_stats": {
      "pages_accessed": 26
    }
  }
}
```

**Quadro Comparativo de Eficiência:**
| Métrica | Paginação com OFFSET | Keyset Pagination | Ganho de Eficiência |
|---|---|---|---|
| **Tempo Real (`r_total_time_ms`)** | `9.00 ms` | `0.02 ms` | **450x mais rápido** |
| **Linhas Examinadas (`r_rows`)** | `90.020` | `20` | **4.500x menos linhas** |
| **Páginas de Buffer Acessadas** | `292` | `26` | **11x menos memória** |

---

## Lições

- **A Chave Primária É "De Graça" no Índice Secundário:** No InnoDB, todo índice secundário contém implicitamente as colunas da chave primária no final de cada entrada. Se sua PK é `id` e você cria um índice em `(cliente_id)`, consultas como `SELECT id FROM pedidos WHERE cliente_id = ?` já são automaticamente *Covering Index*.
- **Estrutura de Índices para Filtro + Order By:** Para zerar o `filesort`, crie o índice respeitando:
  1. Primeiro: colunas da cláusula `WHERE` usadas com igualdade.
  2. Segundo: colunas da cláusula `ORDER BY`.
- **Implementação de Keyset Pagination em APIs:** Em endpoints REST ou GraphQL que realizam scroll infinito ou paginação contínua, substitua parâmetros `?page=450` por `?after_id=90000`. Isso garante latência constante (sub-milissegundo) independentemente da profundidade da busca.

---

## Pegadinhas

- **O Asterisco (`SELECT *`) Destrói o Covering Index:** Se você adicionar uma única coluna desnecessária que não pertença ao índice secundário no `SELECT`, o MariaDB perderá o status de *Covering Index* e será obrigado a fazer o *lookup* completo da linha para cada registro encontrado.
- **`sort_buffer_size` Excessivo Não Resolve Filesort:** Aumentar `sort_buffer_size` para valores altos (ex: 64 MB) não elimina o custo de processamento de CPU do filesort e aumenta o risco de OOM Killer. A verdadeira solução é sempre eliminar o filesort via índice ordenado.
- **Direções Mistas de Ordenação:** Se o índice foi criado em ordem ascendente `(colA, colB)` e a consulta pedir `ORDER BY colA ASC, colB DESC`, o MariaDB **não** conseguirá usar o índice para a ordenação, reintroduzindo o `Using filesort`. No MariaDB 12, use índices descendentes explícitos quando necessário (`CREATE INDEX idx ON tab(colA ASC, colB DESC)`).
