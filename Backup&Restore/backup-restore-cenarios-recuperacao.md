# MariaDB: cenários de recuperação (o playbook do "e agora?")

Um guia de **"aconteceu X, o que eu faço"**. Reúne as receitas de recuperação para os acidentes mais comuns, apontando para o procedimento certo em cada caso. Todos os cenários foram reproduzidos em laboratório.

> Validado em MariaDB 12.3.3 (cluster Galera). Veja também: [estratégia](backup-restore-visao-geral-estrategia.md), [lógico](backup-restore-logico-mariadb-dump.md), [físico](backup-restore-fisico-mariadb-backup.md), [PITR](backup-restore-pitr-binlog.md).

## Para quem é este documento

Estagiários e juniores de plantão. A meta: no momento do aperto, você achar rápido a receita e não travar.

## Tabela de decisão

| Aconteceu... | Receita | Procedimento |
|---|---|---|
| Derrubaram **uma tabela** (`DROP TABLE`) | Restaurar só aquela tabela do dump | Cenário 1 |
| `DELETE`/`UPDATE` errado, você tem binlog | **PITR** até antes do comando | Cenário 2 / [PITR](backup-restore-pitr-binlog.md) |
| Perdeu **um banco** inteiro | Restaurar o dump daquele banco | Cenário 3 |
| Perdeu **um nó** do cluster | Deixar o Galera fazer **SST** | [SST/join](../troubleshooting/troubleshooting-galera-sst-join.md) |
| Perdeu o **cluster inteiro** | Restaurar backup + bootstrap | Cenário 4 |

## Cenário 1 — derrubaram uma tabela

**Sintoma:** `Table 'loja.clientes' doesn't exist`.

**Receita:** restaure **só** aquela tabela a partir de um dump, sem mexer no resto. Testado no lab:

```bash
# (você tinha um dump da tabela)
mariadb loja < /tmp/clientes.sql
```

Resultado real: `clientes` voltou (1000 linhas), `pedidos` **intacto** (5203), e replicou para o db-02. Por isso vale a pena manter **dumps por tabela** dos objetos críticos (ou um dump do banco de onde dá para extrair a tabela com `sed`).

> Se a tabela foi derrubada **há pouco** e você tem binlog, dá para recuperá-la por [PITR](backup-restore-pitr-binlog.md) mesmo sem um dump dela — restaure o base e replay até antes do `DROP`.

## Cenário 2 — `DELETE`/`UPDATE` acidental (o clássico)

**Sintoma:** "rodei um `DELETE` sem `WHERE`" / "atualizei a coluna errada". Em Galera, já replicou para todos os nós.

**Receita:** **PITR** — restaurar o backup base e reaplicar o binlog **até o instante anterior** ao comando errado. Validado no lab (recuperamos 5203 linhas, pulando o `DELETE`):

```bash
mariadb < /tmp/base.sql                                   # volta ao backup
mariadb-binlog --start-position=<pos> \
               --stop-datetime='<logo antes do erro>' \
               /var/lib/mysql/mariadb-bin.NNNNNN | mariadb # reaplica o bom, pula o ruim
```

Passo a passo completo em [PITR](backup-restore-pitr-binlog.md).

> **Primeira reação correta:** **não entre em pânico e não escreva mais nada** na tabela afetada. Quanto menos coisa acontecer depois do erro, mais limpa é a recuperação. Se possível, isole a aplicação.

## Cenário 3 — perdeu um banco inteiro

**Receita:** restaurar o dump daquele banco (ou o backup físico). Testado no lab:

```bash
mariadb -e "DROP DATABASE loja;"      # o desastre
mariadb < /tmp/loja.sql               # restore (dump feito com --databases)
```

Resultado: 1000 clientes e 5000 pedidos de volta. Para bases **grandes**, prefira restaurar de um [backup físico](backup-restore-fisico-mariadb-backup.md) (muito mais rápido que reexecutar `INSERT`s).

## Cenário 4 — perdeu o cluster inteiro (desastre total)

Dois sub-casos:

1. **Os dados nos discos estão ok, só o cluster não sobe** (ex.: todos desligados ao mesmo tempo): não é caso de backup — é **bootstrap**. Veja [recuperação de queda total](../Galera/galera-recuperacao-queda-total.md).
2. **Perdeu os dados** (corrupção, perda das VMs): aí sim é restauração de backup.
   - Restaure um [backup físico](backup-restore-fisico-mariadb-backup.md) (ou lógico) em **um** nó.
   - Se houver binlog, faça [PITR](backup-restore-pitr-binlog.md) até o ponto desejado.
   - Dê o **bootstrap** nesse nó (`galera_new_cluster`) e deixe os outros entrarem por **SST** ([SST/join](../troubleshooting/troubleshooting-galera-sst-join.md)).
   - Valide (`wsrep_cluster_size`, contagens) antes de religar a aplicação.

## Princípios que valem para todo cenário

1. **Pare de piorar.** Isole a aplicação da tabela/banco afetado antes de agir.
2. **Confirme o estrago** antes de recuperar (o que sumiu, desde quando).
3. **Recupere e valide num lugar isolado** quando puder, antes de mexer na produção.
4. **Confira as contagens** depois (linhas, objetos) — restore sem validação não terminou.
5. **Anote o que foi feito** (qual backup, qual ponto no tempo) — serve de pós-morte.

## Checklist pós-recuperação

- [ ] Contagens conferem com o esperado (antes do erro).
- [ ] Em cluster: `wsrep_cluster_size` correto, todos `Synced`, dados replicados.
- [ ] Aplicação reconectada e funcional.
- [ ] Causa do incidente registrada (para não repetir).
- [ ] Backup novo tirado após a recuperação.

## O que NÃO fazer

- **Não escreva mais na tabela afetada** antes de recuperar — atrapalha o PITR.
- **Não restaure por cima da produção "no susto"** sem confirmar o backup e o ponto.
- **Não trate o Galera como backup** — o erro lógico já replicou; só o backup volta no tempo.

## Veja também

- [Estratégia geral](backup-restore-visao-geral-estrategia.md)
- [PITR](backup-restore-pitr-binlog.md)
- [Backup físico](backup-restore-fisico-mariadb-backup.md)
- [Recuperação de queda total do cluster](../Galera/galera-recuperacao-queda-total.md)
