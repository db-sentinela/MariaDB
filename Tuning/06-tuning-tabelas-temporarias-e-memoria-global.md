# 06 — Tabelas Temporárias Internas (Aria), Buffers e Limite Global de RAM

O otimizador de consultas do MariaDB frequentemente precisa materializar resultados intermediários em **tabelas temporárias internas** para resolver operações complexas de `GROUP BY`, `ORDER BY`, `DISTINCT`, `UNION` e subconsultas. Entender o limiar onde essas tabelas migram da memória RAM para o disco, o papel do moderno motor de armazenamento **Aria** e como calcular a pegada total máxima de memória é a última barreira que protege o banco de dados contra o temido **OOM Killer** do Linux.

---

## O que é

### 1. Tabelas Temporárias em Memória vs Disco
Quando uma consulta exige materialização temporária:
- Inicialmente, o MariaDB tenta criar a tabela na memória RAM usando a engine **MEMORY**.
- O tamanho máximo permitido para essa tabela em memória é limitado pelo menor valor entre duas variáveis: `tmp_table_size` e `max_heap_table_size`.
- Se o conjunto de dados em processamento ultrapassar esse limite, o MariaDB realiza o **Spill to Disk**: converte a tabela em memória para uma tabela gravada em disco.

### 2. O Papel do Storage Engine Aria
No MySQL tradicional, tabelas temporárias em disco usavam o antigo motor MyISAM (e mais recentemente o InnoDB). No MariaDB, tabelas temporárias em disco são gerenciadas exclusivamente pela storage engine de alta performance **Aria** (`aria_used_for_temp_tables = ON`). O Aria possui cache de páginas dedicado (`aria_pagecache_buffer_size`), transações *crash-safe* e suporte avançado a linhas de tamanho variável sem o overhead pesado do InnoDB.

### 3. Buffers Globais vs Buffers por Sessão
A memória total de um nó MariaDB é composta por:
1. **Buffers Globais (Alocados no boot/estáticos):**
   - `innodb_buffer_pool_size` (80% da carga principal)
   - `innodb_log_buffer_size` (Redo Log)
   - `aria_pagecache_buffer_size` (Cache do Aria)
   - `key_buffer_size` (Índices MyISAM legados)
2. **Buffers por Conexão (Alocados dinamicamente sob demanda):**
   - `sort_buffer_size` (ordenação `ORDER BY`)
   - `join_buffer_size` (junções sem índice)
   - `read_buffer_size` e `read_rnd_buffer_size` (varreduras)
   - `thread_stack` (pilha de execução por thread)

---

## Por que importa

1. **Impacto Brutal de Desempenho no Spill to Disk:** Uma consulta que resolve um agrupamento em memória demora 5 milissegundos. Se a tabela temporária estourar `tmp_table_size` e for gravada em disco, o tempo de execução sobe para 400 milissegundos devido a operações de I/O e sincronização de blocos.
2. **Prevenção do Kernel OOM Killer:** Se um DBA configurar `sort_buffer_size = 32M` e `join_buffer_size = 32M` pensando em "acelerar queries", em um momento de pico com 500 conexões ativas, a memória requerida apenas para conexões pode alcançar:

$$500 \times (32\text{ MB} + 32\text{ MB} + \text{outros buffers}) \approx 32\text{ GB}$$

Se o servidor possuir apenas 8 GB de RAM, o kernel do Linux matará o processo do MariaDB (`Out of memory: Kill process`).
3. **Calibração do Cache do Aria:** Se `aria_pagecache_buffer_size` for deixado no padrão mínimo de instâncias legadas, consultas analíticas com tabelas temporárias em disco sofrerão contenção no cache do Aria.

---

## Como reproduzir

Execute os testes no nó `mariadb-db-01` para observar as variáveis, forçar o estouro de memória de uma tabela temporária e medir o uso do motor Aria:

### 1. Inspecionar as variáveis de limite temporário e cache Aria
```sql
SHOW VARIABLES WHERE Variable_name IN (
    'tmp_table_size', 
    'max_heap_table_size', 
    'aria_pagecache_buffer_size', 
    'aria_used_for_temp_tables'
);
```

### 2. Verificar os contadores globais de tabelas temporárias
> **Atenção:** Sempre use `SHOW GLOBAL STATUS`, pois `SHOW STATUS` sem o modificador exibe apenas os contadores da conexão atual (que acabou de abrir e estará zerada).

```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
```

### 3. Forçar o Spill to Disk reduzindo o limite na sessão
Crie uma tabela de teste e execute um agrupamento com limite de memória reduzido para apenas 1 KB:

```sql
CREATE DATABASE IF NOT EXISTS lab_tuning;
USE lab_tuning;

CREATE OR REPLACE TABLE t_bench (
    id INT AUTO_INCREMENT PRIMARY KEY,
    dt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    val VARCHAR(50)
) ENGINE=InnoDB;

-- Popula com 2.000 linhas
INSERT INTO t_bench (val) 
SELECT CONCAT('val_', seq) FROM seq_1_to_2000;

-- Reduz propositalmente os buffers temporarios para forcar escrita em disco
SET SESSION tmp_table_size = 1024;
SET SESSION max_heap_table_size = 1024;

-- Query que exige tabela temporaria para agrupamento
SELECT dt, COUNT(DISTINCT val) FROM t_bench GROUP BY dt;
```

### 4. Validar o incremento dos contadores de disco e atividade do Aria
Consulte novamente os status globais do servidor:
```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
SHOW GLOBAL STATUS LIKE 'Aria_pagecache%';
```

---

## O que você vê (saída real)

### Configuração padrão identificada no laboratório:
```text
+----------------------------+-----------+
| Variable_name              | Value     |
+----------------------------+-----------+
| aria_pagecache_buffer_size | 134217728 |
| aria_used_for_temp_tables  | ON        |
| max_heap_table_size        | 16777216  |
| tmp_table_size             | 16777216  |
+----------------------------+-----------+
```
*(No MariaDB 12, `tmp_table_size` e `max_heap_table_size` vêm alinhados em 16 MB, e o cache do Aria em 128 MB).*

### Contadores globais após o teste de estouro de limite:
```sql
SHOW GLOBAL STATUS LIKE 'Created_tmp%';
```
Saída real capturada no laboratório:
```text
+-------------------------+-------+
| Variable_name           | Value |
+-------------------------+-------+
| Created_tmp_disk_tables | 3     |
| Created_tmp_files       | 4     |
| Created_tmp_tables      | 209   |
+-------------------------+-------+
```

### Métricas de atividade do Cache de Páginas do Aria:
```sql
SHOW GLOBAL STATUS LIKE 'Aria_pagecache%';
```
Saída real:
```text
+-----------------------------------+-------+
| Variable_name                     | Value |
+-----------------------------------+-------+
| Aria_pagecache_blocks_not_flushed | 2     |
| Aria_pagecache_blocks_used        | 20    |
| Aria_pagecache_read_requests      | 2195  |
| Aria_pagecache_reads              | 17    |
| Aria_pagecache_write_requests     | 2015  |
| Aria_pagecache_writes             | 3     |
+-----------------------------------+-------+
```
*(Observe `Aria_pagecache_write_requests` e `read_requests` com mais de 2.000 requisições absorvidas pelo cache do Aria durante as operações de materialização).*

---

## Lições

- **Sincronização Obrigatória de Variáveis:** Sempre configure `tmp_table_size` e `max_heap_table_size` com **exatamente o mesmo valor**. Se você definir `tmp_table_size = 64M` e esquecer `max_heap_table_size = 16M`, o MariaDB usará o menor (16 MB), tornando inútil a alteração.
- **A Regra de Ouro da Memória Máxima do MariaDB:**
  Para dimensionar uma máquina sem perigo de OOM Killer, utilize a fórmula:

$$\text{RAM Máxima} = \text{Buffers Globais} + (\text{max\_connections} \times \text{Buffers por Sessão})$$

  Onde:
  - $\text{Buffers Globais} = \text{innodb\_buffer\_pool\_size} + \text{innodb\_log\_buffer\_size} + \text{aria\_pagecache\_buffer\_size}$
  - $\text{Buffers por Sessão} = \text{read\_buffer\_size} + \text{read\_rnd\_buffer\_size} + \text{sort\_buffer\_size} + \text{join\_buffer\_size} + \text{thread\_stack}$

- **Buffers por Sessão Conservadores:** Mantenha `sort_buffer_size` e `join_buffer_size` entre 1 MB e 4 MB. Se uma query analítica específica precisar de mais memória para ordenação em lote, altere o valor apenas na sessão daquela query (`SET SESSION sort_buffer_size = 64M;`), e nunca globalmente.

---

## Pegadinhas

- **A Armadilha do `SHOW STATUS` (Local vs Global):** Executar `SHOW STATUS LIKE 'Created_tmp_disk_tables'` dentro de uma nova sessão de terminal sempre retornará `0`. Estagiários frequentemente acreditam que "não há tabelas indo para disco" porque esqueceram a palavra-chave `GLOBAL`. Sempre audite com `SHOW GLOBAL STATUS`.
- **Colunas `TEXT` e `BLOB` em Consultas:** Historicamente, tabelas que selecionavam colunas `BLOB`/`TEXT` iam imediatamente para o disco porque o motor MEMORY não suportava esses tipos de dados. O MariaDB moderno com motor Aria mitiga parte dessa restrição, mas incluir colunas `TEXT` gigantes em cláusulas `SELECT * ... GROUP BY` continua sendo a principal causa de spill desnecessário para o disco. Selecione apenas as colunas necessárias.
- **Tabelas Temporárias em Disco não Limpas por Falha:** O status `Created_tmp_files` indica descritores de arquivos abertos em `/tmp`. Certifique-se de que a partição `/tmp` do Linux tenha espaço suficiente e não esteja montada com opções restritivas (`noexec,nosuid` podem ser aceitas, mas espaço insuficiente derruba o servidor).
