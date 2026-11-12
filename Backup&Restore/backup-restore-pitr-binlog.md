# MariaDB: PITR — recuperação a um ponto no tempo (binlog)

PITR (*Point-In-Time Recovery*) é recuperar o banco para um **instante exato** — por exemplo, "o estado de 2 segundos antes do `DELETE` errado". É a habilidade que diferencia um DBA, e a única forma de não perder o trabalho feito **depois** do último backup.

> Procedimento **executado e validado** em MariaDB 12.3.3 (cluster Galera, db-01). Saídas reais. Veja também: [estratégia](backup-restore-visao-geral-estrategia.md), [lógico](backup-restore-logico-mariadb-dump.md), [físico](backup-restore-fisico-mariadb-backup.md).

## Para quem é este documento

Estagiários e juniores. A meta: entender a mágica — **backup base + binary log = voltar no tempo** — e saber executar.

## A ideia em uma imagem

- O **backup base** é uma **foto** de um momento (ontem às 2h).
- O **binary log** é o **filme** de tudo que mudou desde a foto.
- **PITR = foto + filme até o frame que você escolher.**

```
backup base ---------[binlog: todas as transações]---------> agora
    (2h)        INSERT  INSERT  UPDATE  <<DELETE errado>>  INSERT
                 \_______ recuperar até AQUI _______/
                          (logo antes do erro)
```

## Pré-requisito: binary log ligado

Sem `log_bin`, **não há PITR**. Em um nó Galera, ligue também `log_slave_updates` para que o binlog capture as escritas vindas de **todos** os nós. Config usada no lab (`/etc/mysql/mariadb.conf.d/70-binlog.cnf`):

```ini
[mysqld]
log_bin           = /var/lib/mysql/mariadb-bin
log_slave_updates = ON
binlog_format     = ROW
expire_logs_days  = 10
```

Confirme:

```sql
SHOW VARIABLES LIKE 'log_bin';         -- ON
SHOW BINARY LOGS;                        -- lista os arquivos mariadb-bin.NNNNNN
```

## O cenário (real, de ponta a ponta)

Reproduzimos o acidente clássico: backup de manhã, trabalho ao longo do dia, e um `DELETE` que apaga tudo.

### 1. Backup base + posição do binlog

```bash
mariadb -e "FLUSH BINARY LOGS"                         # começa um binlog limpo
mariadb-dump --single-transaction --master-data=2 --databases loja > /tmp/base.sql
```

O `--master-data=2` gravou a posição. No lab:

```
arquivo = mariadb-bin.000005   pos = 390
pedidos no backup: 5200
```

### 2. "Trabalho da manhã" (o que NÃO pode ser perdido)

```sql
INSERT INTO loja.pedidos (cliente_id,valor,status) VALUES
 (1,777.77,'manha-pitr'),(2,777.77,'manha-pitr'),(3,777.77,'manha-pitr');
```

Guardamos o **ponto de recuperação** (logo após o trabalho, antes do desastre):

```
ponto de recuperacao = 2026-11-11 23:55:20
```

### 3. O desastre

```sql
DELETE FROM loja.pedidos;        -- apaga TUDO (5203 linhas)
-- pedidos apos desastre: 0
```

Em Galera, isso é ainda pior: o `DELETE` **replicou para todos os nós** na hora. Nenhuma réplica te salva — só o PITR.

### 4. Inspecionar o binlog (ver o que aconteceu)

Antes de recuperar, dá para **ler** o binlog e confirmar os eventos e seus horários:

```bash
mariadb-binlog --base64-output=DECODE-ROWS -v --start-position=390 /var/lib/mysql/mariadb-bin.000005
```

Trecho real (o INSERT da manhã, às 23:55:18, antes do ponto de recuperação):

```
#261111 23:55:18 ... Table_map: `loja`.`pedidos`
### INSERT INTO `loja`.`pedidos`
###   @4='manha-pitr'
### INSERT INTO `loja`.`pedidos`
###   @4='manha-pitr'
### INSERT INTO `loja`.`pedidos`
###   @4='manha-pitr'
```

### 5. A recuperação (2 passos)

**Passo 1 — restaurar o backup base** (volta ao estado do backup, SEM o trabalho da manhã):

```bash
mariadb < /tmp/base.sql
-- após restore base: 5200 pedidos / manha=0
```

**Passo 2 — replay do binlog da posição do backup ATÉ logo antes do desastre:**

```bash
mariadb-binlog --start-position=390 \
               --stop-datetime='2026-11-11 23:55:20' \
               /var/lib/mysql/mariadb-bin.000005 | mariadb
```

### 6. Resultado

```
+-------+-------+
| total | manha |
+-------+-------+
|  5203 |     3 |
+-------+-------+
```

**5203 pedidos** (5200 do backup + 3 da manhã recuperados) e o `DELETE` **não foi reaplicado**. Recuperamos **tudo até o instante anterior ao erro**. 🎯

## Como escolher o ponto de parada

- **`--stop-datetime='AAAA-MM-DD HH:MM:SS'`** — para no primeiro evento com horário ≥ esse. Intuitivo ("recuperar até 23:55:20"). Granularidade de **1 segundo** — deixe folga entre o bom e o ruim.
- **`--stop-position=N`** — para na posição exata do binlog. Mais preciso quando você sabe a posição do comando ruim (localize-a lendo o binlog).
- **`--start-position=N`** — de onde começar (a posição gravada pelo backup base).

> **Vários arquivos de binlog:** se a janela cruza mais de um arquivo, passe-os **em ordem** no mesmo comando: `mariadb-binlog --start-position=390 bin.000005 bin.000006 --stop-datetime='...' | mariadb`.

## Pular apenas o comando ruim (variação)

Se você quer **tudo, menos** o comando ruim (inclusive o que veio depois dele), faça o replay em dois trechos: até **antes** do comando ruim (`--stop-position`) e depois **a partir de depois** dele (`--start-position` no evento seguinte). Assim você "costura" o binlog ignorando só a transação ruim.

## Nota Galera

No lab, o replay foi feito no db-01 e **propagou** para o cluster (as instruções passam pelo wsrep). Em um desastre real de cluster, uma abordagem segura é: restaurar o base + replay em **um** nó isolado, validar, e então reconstituir o cluster a partir dele (bootstrap + SST nos demais — ver [recuperação de queda total](../Galera/galera-recuperacao-queda-total.md)).

## Referência rápida

```bash
# Pré: log_bin=ON (+ log_slave_updates=ON em Galera)
# 1. Backup base com posição
mariadb-dump --single-transaction --master-data=2 --databases db > base.sql
# 2. (desastre acontece)
# 3. Restaurar base
mariadb < base.sql
# 4. Replay até logo antes do erro
mariadb-binlog --start-position=<pos> --stop-datetime='AAAA-MM-DD HH:MM:SS' \
               /var/lib/mysql/mariadb-bin.NNNNNN | mariadb
```

## O que NÃO fazer

- **Não espere o desastre para ligar o binlog.** Sem binlog ligado ANTES, não há como voltar no tempo.
- **Não apague binlogs dentro da janela de retenção** — eles são o "filme". Cuidado com `PURGE BINARY LOGS` e `expire_logs_days` curto demais.
- **Não erre o ponto de parada** — confira lendo o binlog antes de replay. Granularidade de segundo pode incluir/excluir eventos na fronteira.
- **Não replay direto em produção sem validar** — quando possível, recupere e valide em um servidor isolado primeiro.

## Veja também

- [Estratégia geral](backup-restore-visao-geral-estrategia.md)
- [Backup lógico](backup-restore-logico-mariadb-dump.md)
- [Backup físico](backup-restore-fisico-mariadb-backup.md)
- [Cenários de recuperação](backup-restore-cenarios-recuperacao.md)
