# MariaDB: backup físico com mariadb-backup

Backup físico copia os **arquivos** do InnoDB a quente (sem parar o banco). É o método rápido para bases grandes e o padrão para restaurar um cluster. Suporta backup **full** e **incremental**.

> Procedimentos **executados e validados** em MariaDB 12.3.3 / Galera 26.4.27 (db-01). Saídas reais. Veja também: [estratégia](backup-restore-visao-geral-estrategia.md), [lógico](backup-restore-logico-mariadb-dump.md), [PITR](backup-restore-pitr-binlog.md).

## Para quem é este documento

Estagiários e juniores. A meta: tirar um backup físico full e incremental, **prepará-lo** e **restaurá-lo** — e provar que o restore funciona.

> `mariadb-backup` e `mariabackup` são **o mesmo programa** (nomes diferentes). Nesta versão use `mariadb-backup`.

## O fluxo tem sempre 3 fases

1. **`--backup`** — copia os arquivos (a quente).
2. **`--prepare`** — aplica os redo logs, deixando a cópia **consistente e restaurável**.
3. **`--copy-back`** (ou `--move-back`) — coloca os arquivos de volta no `datadir`.

Pular o `--prepare` é o erro nº 1: um backup só copiado **não** sobe.

## 1. Backup full

```bash
mariadb-backup --backup --target-dir=/backup/full --galera-info
```

Saída real (o que você PRECISA ver no fim):

```
[00] ... completed OK!
```

> **`--galera-info`** grava o arquivo `mariadb_backup_galera_info` com a **posição do cluster** (`UUID:seqno`) — importante para restaurar um nó Galera no ponto certo.

Metadados gerados no diretório (nomes desta versão; nas antigas eram `xtrabackup_*`):

```
mariadb_backup_info          # versão, LSN, horário
mariadb_backup_checkpoints   # tipo (full/incremental) e LSN (to_lsn)
mariadb_backup_galera_info   # posição do cluster (com --galera-info)
```

## 2. Backup incremental (só o que mudou)

O incremental copia apenas as páginas alteradas desde um backup anterior (comparando o **LSN**). Economiza tempo e espaço.

```bash
mariadb-backup --backup --target-dir=/backup/inc1 --incremental-basedir=/backup/full
```

No lab: o **full** deu 51M; depois de inserir ~200 linhas, o **incremental** deu 5,7M — bem menor, como esperado. A ligação entre eles é o LSN (`from_lsn` do incremental = `to_lsn` do full), visível em `mariadb_backup_checkpoints`.

## 3. Prepare (deixar restaurável)

> ⚠️ **Diferença importante do Percona:** o `mariadb-backup` do MariaDB **não tem** a opção `--apply-log-only` (isso é do `xtrabackup`). No MariaDB, prepare o full e depois aplique cada incremental.

```bash
# 1. Preparar o full
mariadb-backup --prepare --target-dir=/backup/full
# 2. Aplicar o incremental sobre o full
mariadb-backup --prepare --target-dir=/backup/full --incremental-dir=/backup/inc1
```

Cada etapa tem que terminar com `completed OK!`. Depois disso, o diretório `/backup/full` contém o estado **full + incremental** mesclado, pronto para restaurar.

## 4. Restore e a PROVA de que funciona

Restaurar é `--copy-back` para um `datadir` **vazio**. No lab, para não derrubar o cluster, restauramos para um diretório separado e subimos uma **instância standalone** só para ler os dados:

```bash
# copy-back para um datadir separado
mkdir -p /tmp/restore && chown mysql:mysql /tmp/restore
mariadb-backup --copy-back --target-dir=/backup/full --datadir=/tmp/restore
chown -R mysql:mysql /tmp/restore

# subir uma instância isolada (sem wsrep) só para conferir
mariadbd --no-defaults --user=mysql --datadir=/tmp/restore \
         --socket=/tmp/restore.sock --skip-networking --skip-grant-tables &

mariadb --socket=/tmp/restore.sock -e "SELECT COUNT(*) FROM loja.pedidos;"
```

Resultado real: **pedidos = 5200, clientes = 1000** — exatamente **full (5000) + incremental (200)**. Prova de que o incremental foi aplicado e o backup é restaurável.

> `--no-defaults` evita carregar a config do Galera (senão a instância tentaria entrar no cluster); `--user=mysql` é obrigatório (o `mariadbd` recusa rodar como root).

### Restaurar de verdade em um nó (produção)

Para repor o `datadir` real de um nó:

```bash
sudo systemctl stop mariadb
sudo rm -rf /var/lib/mysql/*
sudo mariadb-backup --copy-back --target-dir=/backup/full
sudo chown -R mysql:mysql /var/lib/mysql
sudo systemctl start mariadb
```

**Em um cluster Galera**, o uso típico do backup físico é **restaurar o cluster inteiro** num desastre total: restaure em **um** nó, dê o bootstrap nele (ver [recuperação de queda total](../Galera/galera-recuperacao-queda-total.md)) e deixe os outros entrarem por **SST**. Para repor **um** nó só, muitas vezes é mais simples apagar o `datadir` e deixar o Galera fazer SST a partir de um nó bom (ver [SST/join](../troubleshooting/troubleshooting-galera-sst-join.md)).

## Backup físico x SST

Os dois usam `mariabackup` por baixo, mas servem a coisas diferentes:

- **SST** reconstrói um nó com o estado **atual** do cluster (não volta no tempo).
- **Backup físico** é uma cópia **guardada** de um momento — é o que te permite voltar a ontem, ou servir de base para [PITR](backup-restore-pitr-binlog.md).

## Referência rápida

```bash
# Full
mariadb-backup --backup --target-dir=/bk/full_$(date +%F) --galera-info
# Incremental
mariadb-backup --backup --target-dir=/bk/inc1 --incremental-basedir=/bk/full_YYYY-MM-DD
# Prepare
mariadb-backup --prepare --target-dir=/bk/full_YYYY-MM-DD
mariadb-backup --prepare --target-dir=/bk/full_YYYY-MM-DD --incremental-dir=/bk/inc1
# Restore
mariadb-backup --copy-back --target-dir=/bk/full_YYYY-MM-DD    # datadir vazio!
```

## O que NÃO fazer

- **Não esqueça o `--prepare`.** Backup só copiado não sobe.
- **Não use `--apply-log-only`** no MariaDB — a opção não existe aqui (é do Percona).
- **Não faça `copy-back` para um `datadir` que não está vazio.** Pare o serviço e limpe antes.
- **Não esqueça o `chown mysql:mysql`** depois do copy-back — arquivos com dono errado impedem o start.

## Veja também

- [Estratégia geral](backup-restore-visao-geral-estrategia.md)
- [Backup lógico](backup-restore-logico-mariadb-dump.md)
- [PITR](backup-restore-pitr-binlog.md)
- [Recuperação de queda total do cluster](../Galera/galera-recuperacao-queda-total.md)
