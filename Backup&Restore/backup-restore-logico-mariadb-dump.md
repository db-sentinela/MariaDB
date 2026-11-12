# MariaDB: backup lógico com mariadb-dump

Backup lógico é um arquivo de texto com os comandos SQL (`CREATE TABLE`, `INSERT`, ...) que reconstroem o banco. É o tipo mais simples e portável — ótimo para entender o conceito, migrar entre versões e restaurar objetos isolados.

> Procedimentos **executados e validados** em MariaDB 12.3.3 (cluster Galera, db-01). Saídas reais. Veja também: [estratégia](backup-restore-visao-geral-estrategia.md), [físico](backup-restore-fisico-mariadb-backup.md), [PITR](backup-restore-pitr-binlog.md).

## Para quem é este documento

Estagiários e juniores. A meta: saber tirar e restaurar um dump com segurança, e entender as opções que fazem diferença.

## Objeto de teste

Banco `loja` (1000 clientes, 5000 pedidos), criado no laboratório para os exemplos.

## Backup: o comando essencial

```bash
mariadb-dump --single-transaction --master-data=2 --databases loja > /tmp/loja.sql
```

Resultado real: arquivo de **303K** para o banco `loja`.

As opções que importam:

- **`--single-transaction`** — tira o dump dentro de uma transação, dando uma **foto consistente** do InnoDB **sem travar** a aplicação. Use sempre (para tabelas InnoDB).
- **`--master-data=2`** — grava no dump a **posição do binary log** no momento do backup, como comentário. É isso que liga o dump ao [PITR](backup-restore-pitr-binlog.md). No dump gerado:
  ```
  -- CHANGE MASTER TO MASTER_LOG_FILE='mariadb-bin.000002', MASTER_LOG_POS=167102;
  ```
- **`--databases loja`** — inclui o `CREATE DATABASE`. Sem `--databases`, o dump traz só as tabelas (você escolhe o banco de destino na hora de restaurar).

Outras muito úteis:

```bash
# Servidor inteiro (todos os bancos)
mariadb-dump --single-transaction --all-databases > /tmp/full.sql

# Com GTID (para posicionar replicação/PITR por GTID)
mariadb-dump --single-transaction --gtid --all-databases > /tmp/full.sql

# Só o esquema (estrutura, sem dados)
mariadb-dump --no-data --databases loja > /tmp/schema.sql

# Só os dados (sem CREATE TABLE)
mariadb-dump --no-create-info --databases loja > /tmp/dados.sql

# Uma tabela só
mariadb-dump --single-transaction loja clientes > /tmp/clientes.sql
```

> **Compacte para guardar:** `mariadb-dump ... | gzip > /tmp/loja.sql.gz`. Dumps de texto comprimem muito bem.

## O que tem dentro do dump

```sql
-- CHANGE MASTER TO MASTER_LOG_FILE='mariadb-bin.000002', MASTER_LOG_POS=167102;
CREATE DATABASE /*!32312 IF NOT EXISTS*/ `loja` ...;
USE `loja`;
DROP TABLE IF EXISTS `clientes`;
CREATE TABLE `clientes` (...);
INSERT INTO `clientes` VALUES (...);
...
-- Dump completed on 2026-11-11 23:50:11
```

Repare: o dump tem `DROP TABLE IF EXISTS` antes de cada `CREATE` — então restaurar **sobrescreve** a tabela. É legível: dá para abrir e inspecionar.

## Restore: simular perda e recuperar

Simulamos a perda do banco inteiro e restauramos:

```bash
# DESASTRE
mariadb -e "DROP DATABASE loja;"          # loja some

# RESTORE
mariadb < /tmp/loja.sql                    # recria tudo a partir do dump
```

Resultado real após o restore: **1000 clientes e 5000 pedidos** de volta.

> **Para onde restaurar:** se o dump foi feito **com** `--databases`, basta `mariadb < arquivo.sql` (ele já sabe o banco). Se foi **sem**, selecione o destino: `mariadb nome_do_banco < arquivo.sql`.

## Restaurar só uma tabela

Dump de uma tabela e restauração sem tocar no resto — testado no lab:

```bash
mariadb-dump --single-transaction loja clientes > /tmp/clientes.sql   # 63K
mariadb -e "DROP TABLE loja.clientes;"                                 # some só clientes
mariadb loja < /tmp/clientes.sql                                       # volta só clientes
```

Resultado: `clientes` de volta (1000 linhas) e `pedidos` **intacto** (5203). Em um dump de banco inteiro, dá para extrair uma tabela com `sed`, mas é bem mais simples ter dumps por tabela quando o objeto é crítico.

## Nota Galera

Rodar o `mariadb < dump` em um nó **replica** o restore para todo o cluster (as instruções passam pelo wsrep). Confirmado no lab: após restaurar no db-01, o db-02 também mostrou os dados. Ou seja, você restaura em **um** nó e o cluster inteiro volta — mas cuidado: um restore grande gera muita replicação.

## Vantagens e limites do backup lógico

**Bom para:** bases pequenas/médias, migração entre versões, restaurar objetos isolados, inspecionar o conteúdo, portabilidade.

**Ruim para:** bases grandes (restore de `INSERT`s é lento — reexecuta tudo), e pesa no servidor durante o dump. Para bases grandes, prefira [backup físico](backup-restore-fisico-mariadb-backup.md).

## Referência rápida

```bash
# Backup
mariadb-dump --single-transaction --master-data=2 --databases loja > loja.sql
mariadb-dump --single-transaction --all-databases | gzip > full_$(date +%F).sql.gz

# Restore
mariadb < loja.sql                       # banco veio no dump (--databases)
mariadb destino < tabelas.sql            # escolher o banco de destino
zcat full.sql.gz | mariadb               # de um .gz
```

## O que NÃO fazer

- **Não tire dump sem `--single-transaction`** em bases InnoDB com tráfego — risco de inconsistência ou de travar tabelas.
- **Não confie em dump sem a posição de binlog** (`--master-data`) se você quer PITR.
- **Não restaure em produção sem conferir** o que o dump contém (ele tem `DROP TABLE` — sobrescreve).

## Veja também

- [Estratégia geral](backup-restore-visao-geral-estrategia.md)
- [Backup físico](backup-restore-fisico-mariadb-backup.md)
- [PITR](backup-restore-pitr-binlog.md)
- [Cenários de recuperação](backup-restore-cenarios-recuperacao.md)
