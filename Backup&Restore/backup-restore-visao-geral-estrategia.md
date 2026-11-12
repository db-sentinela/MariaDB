# MariaDB: backup e restore — visão geral e estratégia

Backup é a apólice de seguro do banco. Este documento dá o **mapa mental**: que tipos existem, quando usar cada um, o que é RPO/RTO, e as regras de ouro — antes de entrar nos procedimentos específicos.

> Base de conhecimento validada em laboratório (MariaDB 12.3.3, cluster Galera). Procedimentos detalhados: [lógico](backup-restore-logico-mariadb-dump.md), [físico](backup-restore-fisico-mariadb-backup.md), [PITR](backup-restore-pitr-binlog.md), [cenários de recuperação](backup-restore-cenarios-recuperacao.md).

## Para quem é este documento

Estagiários e juniores. A meta: entender **por que** existem vários tipos de backup e saber escolher — porque "ter backup" não basta; tem que ser o backup certo, testado.

## A regra que resume tudo

> **Um backup só existe depois que você restaurou com sucesso.** Backup não testado é esperança, não backup. Teste a restauração regularmente.

## Os três tipos (e quando usar)

| Tipo | Ferramenta | O que é | Vantagens | Desvantagens |
|---|---|---|---|---|
| **Lógico** | `mariadb-dump` | Arquivo `.sql` com `CREATE`/`INSERT` | Portável, legível, restaura 1 tabela, migra entre versões | Lento para restaurar bases grandes; pesa no servidor |
| **Físico** | `mariadb-backup` | Cópia dos arquivos do InnoDB (a quente) | Rápido para bases grandes; full + incremental | Mesma major version; menos portável |
| **PITR** | binlog + `mariadb-binlog` | Reaplicar transações até um instante | Recupera **até segundos antes** do erro | Exige `log_bin` ligado e um backup base |

Na prática, eles **se combinam**:

- **Backup base** (lógico ou físico) = a foto de um momento.
- **Binary log** = o "filme" de tudo que aconteceu depois.
- **Backup base + binlog = PITR** = recuperar a qualquer ponto no tempo.

## RPO e RTO (os dois números que definem a estratégia)

- **RPO (Recovery Point Objective):** quanto de dado você aceita perder. Se você faz backup só 1x por dia e não tem binlog, o RPO é de **até 24h**. Com binlog contínuo, o RPO cai para **segundos**.
- **RTO (Recovery Time Objective):** quanto tempo você aceita ficar fora durante a recuperação. Restaurar um dump lógico de 500GB pode levar horas (RTO alto); restaurar um backup físico é bem mais rápido.

> A estratégia de backup é escolhida a partir desses dois números do negócio. "Não posso perder nada e preciso voltar em 10 min" é muito diferente de "perder um dia tudo bem".

## A regra 3-2-1

- **3** cópias dos dados.
- **2** mídias/locais diferentes.
- **1** cópia **fora do local** (offsite).

Backup guardado **na mesma VM** do banco não protege contra perda do servidor. Leve as cópias para fora.

## Backup em um cluster Galera (especificidades)

- **De onde tirar:** faça o backup a partir de um **nó de leitura** (ex.: db-02), para não pesar no nó de escrita. Veja a topologia em [../Galera](../Galera/galera-ha-haproxy-keepalived.md).
- **Backup ≠ alta disponibilidade.** O Galera te dá réplicas vivas (se um nó cai, outro atende), mas **não protege contra erro lógico**: um `DROP TABLE` ou `DELETE` errado **replica para todos os nós instantaneamente**. Só o backup te salva disso.
- **SST não é backup.** O *state transfer* que reconstrói um nó ([SST/join](../troubleshooting/troubleshooting-galera-sst-join.md)) copia o estado **atual** — inclusive o erro que acabou de replicar. Não serve para voltar no tempo.
- **PITR exige `log_bin` ligado** com `log_slave_updates=ON` (para o binlog capturar as escritas vindas de todos os nós). Veja [PITR](backup-restore-pitr-binlog.md).

## Checklist de uma boa estratégia

- [ ] Backup **full** periódico (lógico e/ou físico).
- [ ] **Binary log** ligado para PITR (RPO baixo).
- [ ] Backups **incrementais** entre os fulls (física), para economizar espaço/tempo.
- [ ] Cópia **offsite** (regra 3-2-1).
- [ ] **Retenção** definida (ex.: `binlog_expire_logs_seconds`, rotação dos fulls).
- [ ] **Teste de restauração** periódico e documentado.
- [ ] Backup **monitorado** (alerta se o job falhar — procure sempre o `completed OK!`).

## Por onde começar (ordem de leitura)

1. [Backup lógico](backup-restore-logico-mariadb-dump.md) — o mais simples, bom para entender o conceito.
2. [Backup físico](backup-restore-fisico-mariadb-backup.md) — o padrão para bases grandes e para o cluster.
3. [PITR](backup-restore-pitr-binlog.md) — recuperação a um ponto no tempo (o que diferencia um DBA).
4. [Cenários de recuperação](backup-restore-cenarios-recuperacao.md) — "o que eu faço quando X acontece".

## O que NÃO fazer

- **Não confie em réplica/Galera como backup.** Erro lógico replica; só backup volta no tempo.
- **Não guarde o backup só na mesma máquina do banco.**
- **Não presuma que o backup presta sem testar a restauração.**
- **Não deixe o binlog desligado** se o RPO exigido for baixo — sem binlog, não há PITR.
