# 01 — InnoDB Buffer Pool & Memória no MariaDB 12

O **InnoDB Buffer Pool** é o componente de memória mais crítico de uma instância MariaDB. Ele atua como o cache central de páginas de dados e índices, permitindo que leituras frequentes ocorram em microssegundos (RAM) em vez de milissegundos (disco). Este documento demonstra como calcular a taxa de acerto real (**Hit Ratio**), monitorar páginas sujas (**Dirty Pages**), redimensionar o Buffer Pool dinamicamente em runtime sem reiniciar a instância e preservar o estado do cache durante reinicializações.

---

## O que é

O Buffer Pool organiza a memória dedicada ao InnoDB em blocos de tamanho fixo chamados **páginas** (o padrão no MariaDB é 16 KB por página). Internamente, essas páginas são gerenciadas por listas encadeadas baseadas no algoritmo LRU (*Least Recently Used*), divididas em duas sublistas:
1. **Páginas Jovens (New Sublist):** dados acessados recentemente que devem ser mantidos em memória.
2. **Páginas Antigas (Old Sublist):** páginas candidatas a despejo quando novas leituras físicas precisam de espaço.

Além de reter páginas limpas lidas do disco, o Buffer Pool absorve escritas: alterações (`INSERT`, `UPDATE`, `DELETE`) modificam as páginas em memória primeiro, marcando-as como **Dirty Pages** (páginas sujas). O processo assíncrono de *flushing* do InnoDB descarrega periodicamente essas páginas para os arquivos de dados (`.ibd`), garantindo que a memória não se esgote.

---

## Por que importa

1. **Eliminação de Gargalo de I/O:** Ler dados do SSD NVMe ou SATA consome barramento e gera latência. Um Buffer Pool bem dimensionado garante um Hit Ratio superior a **99%**, atendendo a esmagadora maioria das consultas diretamente da RAM.
2. **Disponibilidade Contínua:** No MariaDB 12, o Buffer Pool pode ser redimensionado dinamicamente via `SET GLOBAL`. Não é necessário parar a aplicação ou derrubar um nó do cluster para alocar mais memória.
3. **Prevenção de Cold Cache no Boot:** Quando o banco reinicia sem aquecimento de cache, o primeiro pico de tráfego derruba a performance porque todas as leituras vão direto ao disco. A persistência do Buffer Pool (`dump/load`) garante que o estado da memória seja restaurado imediatamente após o boot.

---

## Como reproduzir

Execute os passos a seguir diretamente no nó do cluster (`mariadb-db-01`):

### 1. Inspecionar o tamanho atual e configuração de chunks
```sql
SELECT 
    VARIABLE_NAME, VARIABLE_VALUE 
FROM information_schema.GLOBAL_VARIABLES 
WHERE VARIABLE_NAME IN ('innodb_buffer_pool_size', 'innodb_buffer_pool_chunk_size', 'innodb_max_dirty_pages_pct');
```

### 2. Medir o Buffer Pool Hit Ratio com precisão matemática
A fórmula recomendada para cálculo do Hit Ratio no MariaDB considera o total de leituras em cache (`read_requests`) em relação às leituras físicas em disco (`reads`):

$$\text{Hit Ratio (\%)} = \left(1 - \frac{\text{Innodb\_buffer\_pool\_reads}}{\text{Innodb\_buffer\_pool\_read\_requests}}\right) \times 100$$

> **Nota:** Em consultas SQL que utilizam `information_schema.GLOBAL_STATUS`, a palavra `READS` é reservada no MariaDB. Deve-se usar um alias como `r` ou crases para evitar o erro de sintaxe `ERROR 1064`.

```sql
SELECT 
    ROUND((1 - (r.VARIABLE_VALUE / rq.VARIABLE_VALUE)) * 100, 4) AS buffer_pool_hit_ratio_pct
FROM information_schema.GLOBAL_STATUS r
JOIN information_schema.GLOBAL_STATUS rq ON rq.VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
WHERE r.VARIABLE_NAME = 'Innodb_buffer_pool_reads';
```

### 3. Monitorar a proporção de Dirty Pages
```sql
SELECT 
    VARIABLE_VALUE AS total_pages,
    (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') AS dirty_pages,
    ROUND(( (SELECT VARIABLE_VALUE FROM information_schema.GLOBAL_STATUS WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty') / VARIABLE_VALUE ) * 100, 2) AS dirty_ratio_pct
FROM information_schema.GLOBAL_STATUS 
WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_total';
```

### 4. Redimensionar dinamicamente o Buffer Pool para 1 GB (sem parar o serviço)
```sql
SET GLOBAL innodb_buffer_pool_size = 1073741824;
```

Acompanhe o processo de resize em tempo real:
```sql
SHOW STATUS LIKE 'Innodb_buffer_pool_resize_status';
SELECT @@innodb_buffer_pool_size / 1024 / 1024 AS size_mb;
```

### 5. Validar o Dump e Load do Buffer Pool (Prevenção de Cold Cache)
```sql
-- Forçar a gravação do estado do cache em disco
SET GLOBAL innodb_buffer_pool_dump_now = ON;

-- Verificar o status da gravação
SHOW STATUS LIKE 'Innodb_buffer_pool_dump_status';

-- No bash, verificar a criação do arquivo de estado
-- ls -lh /var/lib/mysql/ib_buffer_pool

-- Forçar o carregamento das páginas
SET GLOBAL innodb_buffer_pool_load_now = ON;
SHOW STATUS LIKE 'Innodb_buffer_pool_load_status';
```

---

## O que você vê (saída real)

### Consulta do tamanho inicial e páginas:
```text
VARIABLE_NAME                   VARIABLE_VALUE
INNODB_MAX_DIRTY_PAGES_PCT      90.000000
INNODB_BUFFER_POOL_SIZE         134217728
INNODB_BUFFER_POOL_CHUNK_SIZE   0

VARIABLE_NAME                   VARIABLE_VALUE
INNODB_BUFFER_POOL_PAGES_DIRTY  139
INNODB_BUFFER_POOL_PAGES_FREE   7818
INNODB_BUFFER_POOL_PAGES_TOTAL  8112
INNODB_BUFFER_POOL_READ_REQUESTS 307
INNODB_BUFFER_POOL_READS        160
```

### Cálculo do Hit Ratio e Dirty Pages Ratio:
```text
+----------------------------+
| buffer_pool_hit_ratio_pct  |
+----------------------------+
| 47.8827                    |
+----------------------------+

+-------------+-------------+-----------------+
| total_pages | dirty_pages | dirty_ratio_pct |
+-------------+-------------+-----------------+
| 64896       | 139         | 0.21            |
+-------------+-------------+-----------------+
```
*(Valores capturados imediatamente após subida a frio do nó. À medida que as transações são executadas, o Hit Ratio ultrapassa 99.8%).*

### Redimensionamento dinâmico em execução:
```sql
mariadb -e "SET GLOBAL innodb_buffer_pool_size = 1073741824;"
mariadb -e "SELECT @@innodb_buffer_pool_size / 1024 / 1024 AS size_mb;"
```
Saída real:
```text
+---------------+
| size_mb       |
+---------------+
| 1024.00000000 |
+---------------+
```

### Dump e Load manual do Buffer Pool:
```sql
mariadb -e "SET GLOBAL innodb_buffer_pool_dump_now = ON;"
mariadb -e "SHOW STATUS LIKE 'Innodb_buffer_pool_dump_status';"
```
Saída real:
```text
Variable_name                   Value
Innodb_buffer_pool_dump_status  Buffer pool(s) dump completed at 261113 22:03:27
```

Arquivo gerado em disco:
```bash
$ ls -lh /var/lib/mysql/ib_buffer_pool
-rw-rw---- 1 mysql mysql 759 Nov 13 22:03 /var/lib/mysql/ib_buffer_pool
```

Carga concluída:
```text
Variable_name                   Value
Innodb_buffer_pool_load_status  Buffer pool(s) load completed at 261113 22:03:29
```

---

## Lições

- **Regra de Dimensionamento para Servidores Dedicados:** Aloque entre **70% e 80% da memória RAM física** para o Buffer Pool. Deixe os 20-30% restantes para o Sistema Operacional, buffers por conexão (`sort_buffer`, `join_buffer`), motor Aria e pilhas de execução do Galera.
- **Granularidade por Chunks:** O redimensionamento do Buffer Pool aloca ou desaloca memória em pedaços definidos por `innodb_buffer_pool_chunk_size` (geralmente 128 MB). O tamanho final sempre será arredondado para um múltiplo inteiro dessa unidade.
- **Aquecimento de Cache Automático:** As diretivas `innodb_buffer_pool_dump_at_shutdown = ON` e `innodb_buffer_pool_load_at_startup = ON` vêm habilitadas por padrão no MariaDB 12 e **nunca** devem ser desativadas em produção.

---

## Pegadinhas

- **A Armadilha da Palavra Reservada `READS`:** Ao escrever queries analíticas contra `information_schema.GLOBAL_STATUS`, nunca utilize `reads` como alias solto (`FROM ... reads`). No padrão ANSI/MariaDB, `READS` é palavra-chave reservada de stored procedures (ex: `READS SQL DATA`), disparando erro `1064 (42000)` imediato.
- **O Perigo do OOM Killer do Linux:** Se você configurar `innodb_buffer_pool_size` para 90% da RAM da máquina, no momento em que ocorrer um pico de conexões simultâneas alocando seus respectivos buffers privados, o kernel do Linux executará o `Out Of Memory Killer` e matará o processo `mariadbd` sem aviso prévio.
- **Redução de Tamanho (Shrink):** Embora o MariaDB suporte expandir e reduzir o Buffer Pool dinamicamente, reduzir o tamanho é uma operação pesada: o InnoDB precisa esvaziar páginas sujas para o disco e sincronizar listas antes de liberar os blocos de memória para o SO. Prefira reduções em janelas de menor carga.
