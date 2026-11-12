# DB Sentinel | MariaDB — Backup & Restore

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração de bancos de dados.

Esta subpasta cobre **backup e restauração de MariaDB** — lógico, físico, PITR e os cenários de recuperação de desastre. É um dos temas mais críticos: alta disponibilidade (Galera) protege contra falha de hardware, mas **só o backup protege contra erro lógico** (um `DROP`/`DELETE` errado replica para todos os nós). Procedimentos independentes de clientes, sem dados reais, validados em laboratório.

## Procedimentos

| Procedimento | Descrição | Status |
|---|---|---|
| [Visão geral e estratégia](backup-restore-visao-geral-estrategia.md) | Tipos de backup, RPO/RTO, regra 3-2-1, especificidades do Galera, checklist | Validado |
| [Backup lógico (mariadb-dump)](backup-restore-logico-mariadb-dump.md) | Dump consistente, `--master-data`, restore de banco e de tabela única | Validado |
| [Backup físico (mariadb-backup)](backup-restore-fisico-mariadb-backup.md) | Full + incremental, prepare, copy-back e prova de restore | Validado |
| [PITR (point-in-time recovery)](backup-restore-pitr-binlog.md) | Binary log + replay até o instante anterior ao erro | Validado |
| [Cenários de recuperação](backup-restore-cenarios-recuperacao.md) | Playbook "aconteceu X, o que faço": drop de tabela, delete acidental, perda de banco/cluster | Validado |

## Ambiente de homologação

| Item | Versão |
|---|---|
| Sistema operacional | Ubuntu 26.04.1 LTS |
| MariaDB Server | 12.3.3 |
| Ferramentas | `mariadb-dump`, `mariadb-backup`, `mariadb-binlog` |
| Topologia | nó do cluster Galera (ver `../Galera`) |
| Virtualização | Oracle VirtualBox |

## A regra de ouro

> **Um backup só existe depois que você restaurou com sucesso.** Todo procedimento aqui foi fechado com uma **restauração real** e conferência de contagem — não só a geração do backup.

## Padrão dos documentos

- Conceito → comando → **saída real** do laboratório → interpretação.
- Cada backup é acompanhado da sua **restauração validada**.
- Como **resolver** e o que **NÃO fazer**.
- Especificidades de Galera destacadas (erro lógico replica; SST não é backup; PITR exige `log_bin`).

## Estrutura

```
backup-restore/
├── README.md
├── backup-restore-visao-geral-estrategia.md
├── backup-restore-logico-mariadb-dump.md
├── backup-restore-fisico-mariadb-backup.md
├── backup-restore-pitr-binlog.md
└── backup-restore-cenarios-recuperacao.md
```

## Objetos de teste

Os cenários usam um banco descartável `loja` (tabelas `clientes` e `pedidos`), criado e removido durante os testes. Para o PITR, o binary log é habilitado no nó de teste. Nenhum objeto de teste fica no servidor após a publicação.

## Como usar

1. Leia a [visão geral](backup-restore-visao-geral-estrategia.md) para escolher a estratégia (RPO/RTO).
2. Implemente o backup base ([lógico](backup-restore-logico-mariadb-dump.md) ou [físico](backup-restore-fisico-mariadb-backup.md)).
3. Ligue o binary log e domine o [PITR](backup-restore-pitr-binlog.md) para RPO baixo.
4. Tenha o [playbook de recuperação](backup-restore-cenarios-recuperacao.md) à mão para o dia do incidente.
5. **Teste a restauração** periodicamente — é o único jeito de saber que o backup presta.

## Aviso

Os procedimentos foram validados em laboratório e incluem ações destrutivas controladas (`DROP`, `DELETE`, apagar datadir). Execute em homologação antes de produção, e nunca restaure por cima de produção sem confirmar o backup e o ponto de recuperação.

## Mantenedor

DB Sentinel Consultoria em Tecnologia da Informacao Ltda.
