# MariaDB: queries lentas — slow log, EXPLAIN e índices

Como achar a query que está deixando o banco lento, entender **por que** ela é lenta e corrigir. O caso mais comum — de longe — é **falta de índice**, e este documento mostra o diagnóstico de ponta a ponta com um exemplo real.

> Procedimento **executado e validado** em MariaDB 12.3.3 (nó do cluster Galera, db-01), com uma tabela de **500.000 linhas**. Saídas reais. Veja também: [onde procurar](troubleshooting-diagnostico-onde-procurar.md).

## Para quem é este documento

Estagiários e juniores. A meta: dado "o sistema está lento", você achar a query culpada e, em muitos casos, resolver com um `CREATE INDEX`.

## Objeto de teste

```sql
CREATE TABLE lab_tshoot.eventos (
  id INT PRIMARY KEY,
  usuario_id INT NOT NULL,
  status VARCHAR(20) NOT NULL,
  criado_em DATETIME NOT NULL,
  payload VARCHAR(100)
) ENGINE=InnoDB;
-- 500.000 linhas geradas com o sequence engine:
-- INSERT ... SELECT seq, ... FROM seq_1_to_500000;
```

## 1. Ligar o slow query log (achar o que é lento)

O slow log registra toda query que passar de um tempo (`long_query_time`). Para caçar, baixe o limiar:

```sql
SET GLOBAL slow_query_log      = ON;
SET GLOBAL slow_query_log_file = '/tmp/lab_slow.log';  -- padrão: <datadir>/<host>-slow.log
SET GLOBAL long_query_time     = 0.01;                 -- 10 ms (em produção, 0.5 a 1s)
```

> Isso vale em runtime (some no restart). Para ficar permanente, coloque no `.cnf`. Lembre: nesta instalação o log de erro é o `journalctl -u mariadb`, mas o slow log vai para **arquivo**.

Rode uma query que varre a tabela toda (um `LIKE '%...%'` nunca usa índice):

```sql
SELECT COUNT(*) FROM eventos WHERE payload LIKE '%payload-4999%';
```

A entrada real que apareceu no `/tmp/lab_slow.log`:

```
# Query_time: 0.117668  Lock_time: 0.000071  Rows_sent: 1  Rows_examined: 500000
SELECT COUNT(*) FROM eventos WHERE payload LIKE '%payload-4999%';
```

Como ler:

- **`Query_time: 0.117668`** — levou 117 ms.
- **`Rows_examined: 500000`** — teve que **olhar 500 mil linhas** para devolver 1 (`Rows_sent: 1`). Essa desproporção (examinou muito, devolveu pouco) é a **assinatura de falta de índice**.

Para analisar o arquivo agregando as piores queries:

```bash
sudo mariadb-dumpslow -s t /tmp/lab_slow.log   # ordena por tempo
```

## 2. Entender por que é lenta — `EXPLAIN`

O `EXPLAIN` mostra o **plano** sem executar. Numa query sem índice:

```sql
EXPLAIN SELECT COUNT(*), MAX(criado_em) FROM eventos WHERE usuario_id = 5000\G
```

Saída real (ANTES do índice):

```
        table: eventos
         type: ALL            <- full table scan (o pior caso)
possible_keys: NULL           <- nenhum índice utilizável
          key: NULL
         rows: 498829         <- estimativa de linhas lidas: a tabela inteira
        Extra: Using where
```

Os sinais de alerta no `EXPLAIN`:

- **`type: ALL`** — *full table scan*. Lê tudo. (A ordem do melhor para o pior: `const` → `eq_ref` → `ref` → `range` → `index` → `ALL`.)
- **`key: NULL`** — nenhum índice usado.
- **`rows`** alto — quantas linhas o otimizador acha que vai ler.

Confirme o tempo e as linhas **reais** com `ANALYZE` (executa e mede):

```sql
ANALYZE FORMAT=JSON SELECT COUNT(*), MAX(criado_em) FROM eventos WHERE usuario_id = 5000;
```

Valor real medido: **`r_total_time_ms` ≈ 47,9** e **`r_rows`: 500000** (leu as 500 mil linhas).

## 3. Corrigir — criar o índice

```sql
CREATE INDEX idx_usuario ON eventos(usuario_id);
```

`EXPLAIN` da **mesma** query, agora:

```
        table: eventos
         type: ref             <- usa índice (busca direta)
possible_keys: idx_usuario
          key: idx_usuario
      key_len: 4
          ref: const
         rows: 35              <- lê ~35 linhas, não 500 mil
        Extra:
```

E o tempo real:

| | Antes (sem índice) | Depois (com índice) |
|---|---|---|
| `type` (EXPLAIN) | `ALL` (full scan) | `ref` (busca por índice) |
| Linhas lidas (`r_rows`) | 500.000 | 35 |
| Tempo real (`r_total_time_ms`) | ~47,9 ms | ~0,09 ms |

Ou seja: de varrer 500 mil linhas para ler 35 — **centenas de vezes mais rápido**, mudando só o plano de acesso.

## 4. Como escolher o índice

- Indexe as colunas usadas em **`WHERE`**, **`JOIN ... ON`** e **`ORDER BY`**.
- Para filtros por várias colunas, use **índice composto** na ordem dos filtros mais seletivos (ex.: `INDEX(usuario_id, status)`).
- **`LIKE '%texto%'`** (curinga à esquerda) **não usa índice** — precisa de outra estratégia (full-text, ou repensar a query).
- Funções sobre a coluna (`WHERE YEAR(criado_em)=2026`) também **anulam** o índice; prefira `WHERE criado_em >= '2026-01-01' AND criado_em < '2027-01-01'`.

## Nota Galera

`CREATE INDEX`/`ALTER` em um cluster roda por **TOI** e replica para todos os nós — e pode **abortar transações** da aplicação que estejam na tabela (ver [locks e bloqueios](troubleshooting-locks-e-bloqueios.md)). Crie índices em janela controlada. Em tabelas grandes, avalie ferramentas de DDL online.

## Roteiro rápido

```sql
-- 1. Ligar o caçador
SET GLOBAL slow_query_log=ON; SET GLOBAL long_query_time=0.5;
-- 2. Ver o plano da query suspeita
EXPLAIN <sua query>\G          -- procure type:ALL, key:NULL, rows alto
-- 3. Medir de verdade
ANALYZE FORMAT=JSON <sua query>;
-- 4. Indexar e re-medir
CREATE INDEX idx_x ON tabela(coluna);
```

## O que NÃO fazer

- **Não crie índice em tudo.** Índice acelera leitura, mas pesa na escrita e ocupa disco. Crie os que o `EXPLAIN`/slow log justificam.
- **Não confie só na estimativa do `EXPLAIN`.** Confirme com `ANALYZE` (números reais).
- **Não deixe o slow log com `long_query_time=0` em produção** — ele registra tudo e o arquivo explode. Use para caçar, depois volte para 0.5–1s.

## Veja também

- [Onde procurar / diagnóstico geral](troubleshooting-diagnostico-onde-procurar.md)
- [Locks e bloqueios](troubleshooting-locks-e-bloqueios.md)
