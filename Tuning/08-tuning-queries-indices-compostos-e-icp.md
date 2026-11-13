# 08 — Estratégias de Indexação: Índices Compostos, Leftmost Prefix e ICP

Criar índices sem compreender a mecânica interna da árvore B-Tree do InnoDB gera bancos lentos e inflados. Este documento aborda as três regras mestras da indexação avançada no MariaDB: o design de **Índices Compostos**, o respeito à **Leftmost Prefix Rule** (regra do prefixo mais à esquerda) e o poder do **Index Condition Pushdown (ICP)** para reduzir drasticamente o acesso a páginas de dados.

---

## O que é

### 1. Índices Compostos e a Leftmost Prefix Rule
Um índice composto é uma estrutura de árvore B-Tree ordenada lexicograficamente por múltiplas colunas: `(colA, colB, colC)`.
- Os registros são ordenados primeiro por `colA`.
- Para valores idênticos de `colA`, os registros são ordenados por `colB`.
- Para valores idênticos de `colB`, são ordenados por `colC`.

Por consequência dessa física de dados, a **Leftmost Prefix Rule** determina que o índice só pode ser utilizado para acelerar a busca se a consulta filtrar a **primeira coluna do índice** (`colA`). Uma busca que filtre apenas `colB` ou `colC` **não pode** navegar pela árvore e resultará em uma varredura completa da tabela (`type: ALL`).

### 2. Index Condition Pushdown (ICP)
Sem o ICP (comportamento tradicional):
1. O InnoDB usa a primeira coluna do índice para localizar as entradas na B-Tree.
2. Para cada entrada encontrada, o InnoDB busca a linha inteira no índice primário (*clustered index*).
3. A linha completa é enviada para o servidor MariaDB, que avalia os filtros restantes no `WHERE`.

Com o **Index Condition Pushdown (`Using index condition`)**:
O MariaDB "empurra" (*pushes down*) a avaliação de partes da cláusula `WHERE` para dentro da própria engine de armazenamento InnoDB. O InnoDB valida a condição diretamente sobre os nós do índice secundário. Se o registro não atender ao critério, o InnoDB **descarta a entrada imediatamente sem acessar a linha na tabela primária**.

---

## Por que importa

1. **Economia Massiva de Random I/O:** Sem o ICP, se uma busca por `cliente_id = 500` encontrar 10.000 pedidos e apenas 2 tiverem `codigo_rastreio LIKE 'BR%XP'`, o InnoDB faria 10.000 leituras no índice primário para descartar 9.998 linhas no servidor. Com o ICP, ele lê as 10.000 chaves no índice secundário (que é compacto), filtra as 2 linhas e faz apenas **duas** leituras na tabela.
2. **Consolidação de Índices:** Compreender a Leftmost Prefix Rule permite substituir múltiplos índices redundantes (ex: um índice em `cliente_id` e outro em `(cliente_id, criado_em)`) por um único índice composto inteligente, economizando espaço em disco e acelerando os comandos de `INSERT`/`UPDATE`.
3. **Prevenção da Supressão de Índices por Funções:** Aplicar funções como `YEAR(data)`, `DATE(data)` ou `LOWER(nome)` em colunas indexadas desativa a busca na B-Tree, forçando um full table scan mesmo que exista um índice perfeito.

---

## Como reproduzir

Execute os testes no nó `mariadb-db-01` usando a tabela `pedidos`:

### 1. Criar índice composto em `(cliente_id, codigo_rastreio)`
```sql
USE lab_tuning;
CREATE INDEX idx_ped_cli_cod ON pedidos(cliente_id, codigo_rastreio);
```

### 2. Validar o Index Condition Pushdown (ICP) em ação
Execute uma busca filtrando por igualdade na primeira coluna e por padrão de texto na segunda:
```sql
EXPLAIN SELECT * FROM pedidos 
WHERE cliente_id = 500 AND codigo_rastreio LIKE 'BR%XP'\G
```
*(Observe o indicador `Extra: Using index condition` e `key_len: 207`).*

### 3. Demonstrar a violação da Leftmost Prefix Rule
Tente buscar exclusivamente pela segunda coluna do índice composto:
```sql
EXPLAIN SELECT * FROM pedidos 
WHERE codigo_rastreio = 'BR000000500XP'\G
```
*(Observe que o índice é totalmente ignorado: `key: NULL` e `type: ALL`).*

### 4. Supressão de índice por função no `WHERE`
Crie um índice na coluna temporal e teste o filtro com função versus range:
```sql
CREATE INDEX idx_pedidos_data ON pedidos(criado_em);

-- 4A: Consulta com funcao (Desativa o indice)
EXPLAIN SELECT * FROM pedidos WHERE YEAR(criado_em) = 2026\G

-- 4B: Consulta corrigida com Range puro e seletivo (Ativa o indice)
EXPLAIN SELECT * FROM pedidos WHERE criado_em >= '2026-11-01'\G
```

---

## O que você vê (saída real)

### 1. ICP em execução (`Using index condition`):
```text
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: pedidos
         type: range
possible_keys: idx_ped_cli_cod
          key: idx_ped_cli_cod
      key_len: 207
          ref: NULL
         rows: 1
        Extra: Using index condition
```
**Análise:** O otimizador utilizou o índice composto inteiro (`key_len: 207`), navegando pelo cliente e filtrando o padrão de rastreio dentro do InnoDB, resultando em apenas 1 linha examinada (`rows: 1`).

### 2. Violação da regra do prefixo mais à esquerda:
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
**Análise:** Como a consulta não informou `cliente_id` (a coluna líder do índice), o MariaDB foi forçado a varrer a tabela inteira (`rows: 100116`).

### 3. Supressão de índice por função (`YEAR`):
```text
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: pedidos
         type: ALL
possible_keys: idx_pedidos_data
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 100116
        Extra: Using where
```
**Análise:** Mesmo existindo `idx_pedidos_data` em `possible_keys`, o otimizador não consegue calcular os valores inversos da função `YEAR()` para navegar na árvore, abandonando o índice e fazendo scan completo.

### 4. Correção usando condição sargável (Range Puro):
```text
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: pedidos
         type: range
possible_keys: idx_pedidos_data
          key: idx_pedidos_data
      key_len: 5
          ref: NULL
         rows: 3561
        Extra: Using index condition
```
**Análise:** Ao substituir a função por `criado_em >= '2026-11-01'`, a consulta tornou-se **Sargable** (*Search Argument Able*). O índice foi ativado em modo `range`, processando apenas 3.561 linhas.

---

## Lições

- **Regra de Ordenação das Colunas no Índice Composto:**
  1. Primeiro coloque as colunas usadas em **comparações de igualdade** (`=` ou `IS NULL`).
  2. Depois coloque as colunas usadas em **filtros de intervalo** (`BETWEEN`, `>`, `<`, `LIKE 'abc%'`).
  3. Por fim, adicione colunas necessárias para ordenação (`ORDER BY`) ou para fechar índice de cobertura.
- **Consultas Sargables (Search Argument Able):** Nunca envolva a coluna do banco dentro de uma função no lado esquerdo do operador. Sempre transforme o lado direito:
  - ❌ `WHERE DATE(criado_em) = '2026-11-10'` (Destrói o índice)
  - ✔️ `WHERE criado_em >= '2026-11-10 00:00:00' AND criado_em < '2026-11-11 00:00:00'` (Preserva o índice)
- **Status do ICP:** No MariaDB 12, o ICP vem ligado por padrão (`optimizer_switch = 'index_condition_pushdown=on'`). Caso precise validar seu impacto em benchmarks, pode ser desativado temporariamente na sessão via `SET optimizer_switch = 'index_condition_pushdown=off'`.

---

## Pegadinhas

- **O Mito do "Curinga no Início" (`LIKE '%XP'`):** O ICP e a B-Tree suportam apenas prefixos fixos (`LIKE 'BR%'`). Se o padrão de busca iniciar com `%` (ex: `LIKE '%001'`), o índice B-Tree não pode ser utilizado para posicionamento na árvore, degradando para varredura completa.
- **Tipos de Dados Divergentes e Cast Implícito:** Se uma coluna for `VARCHAR` e a consulta comparar com número sem aspas (`WHERE codigo_rastreio = 12345`), o MariaDB converterá silenciosamente cada linha da tabela aplicando a função `CAST()`, desativando o índice sem emitir nenhum erro de sintaxe.
- **Limiar de Seletividade do Otimizador:** Se uma condição de range selecionar mais de 20% a 30% da tabela, o otimizador de custos pode ignorar o índice deliberadamente e optar por `ALL` (Full Table Scan). Isso ocorre porque ler metade da tabela via índice causa um número massivo de acessos aleatórios que custa mais caro do que uma leitura sequencial do disco.
