# MariaDB Galera: um nó não entra — SST, IST e state transfer

Quando um nó do cluster não sincroniza, o problema quase sempre é o **state transfer** (a cópia de dados que coloca o nó em dia). Este documento explica IST x SST, mostra um SST completo **forçado em laboratório**, e lista onde olhar quando dá errado.

> Procedimento **executado e validado** em MariaDB 12.3.3 / Galera 26.4.27. Saídas reais. Veja também: [instalação do cluster](../Galera/galera-ha-haproxy-keepalived.md), [recuperação de queda total](../Galera/galera-recuperacao-queda-total.md), [monitoramento](../Galera/galera-monitoramento-observabilidade.md).

## Para quem é este documento

Estagiários e juniores. A meta: entender como um nó "entra" no cluster e saber diagnosticar quando ele fica preso em `Joining` e não chega a `Synced`.

## IST x SST (a diferença que explica tudo)

Quando um nó entra no cluster, ele precisa ficar idêntico aos outros. Há dois caminhos:

- **IST (Incremental State Transfer):** o nó estava fora por **pouco tempo** e os writesets que ele perdeu ainda estão no **gcache** do doador. Manda só o delta — rápido, segundos.
- **SST (State Snapshot Transfer):** o nó é **novo** ou ficou fora tempo demais (o gcache já não tem o que ele perdeu). Copia o **banco inteiro** via `mariabackup` de um nó doador. Mais demorado, proporcional ao tamanho dos dados.

> O tamanho do `gcache` (`gcache.size=512M` neste lab) decide a fronteira: quanto maior, mais tempo um nó pode ficar fora e ainda voltar por IST em vez de SST.

Estados que um nó atravessa (veja em `wsrep_local_state_comment`):

```
Joining  ->  Joined  ->  (Donor/Desynced no doador)  ->  Synced
```

## Forçando um SST completo (demonstração real)

Para ver um SST do zero, apagamos o datadir de um nó e o iniciamos — sem dados locais, ele é obrigado a copiar tudo.

> ⚠️ Destrutivo no nó alvo, mas **recuperável por projeto**: é exatamente isso que o SST conserta. Nunca faça no nó errado nem em todos ao mesmo tempo.

```bash
# No db-02 (o nó a reconstruir)
sudo systemctl stop mariadb
sudo rm -rf /var/lib/mysql
sudo mkdir /var/lib/mysql
sudo chown mysql:mysql /var/lib/mysql
sudo chmod 700 /var/lib/mysql
sudo systemctl start mariadb      # vai disparar o SST
```

Enquanto o db-02 recebe os dados, observamos o **doador (db-01)** mudar de estado. Saída real, de 2 em 2 segundos:

```
t=1s   db-01 estado: Synced
t=3s   db-01 estado: Donor/Desynced   <- db-01 virou DOADOR do SST
t=5s   db-01 estado: Donor/Desynced
t=7s   db-01 estado: Donor/Desynced
t=9s   db-01 estado: Synced           <- SST terminou, doador voltou ao normal
```

No **journal do db-02**, a conclusão do transfer:

```
WSREP: State transfer from 2.0 (mariadb-db-01) complete.
WSREP: Receiving IST... 100.0% (7/7 events) complete.
```

> Note que, após a cópia física (SST via mariabackup), ainda houve um **pequeno IST** para aplicar os últimos eventos que ocorreram durante a cópia. É normal: SST traz o grosso, o IST final acerta o ponteiro.

Estado final — reconstruído por completo:

```
wsrep_cluster_size   3
wsrep_cluster_status Primary
db-02 linhas apos SST: 500000     <- todas as 500 mil linhas voltaram
```

## O doador fica disponível durante o SST?

Com `wsrep_sst_method = mariabackup` (o deste lab), **sim**: o `mariabackup` é um backup a quente, então o doador (db-01) **continua atendendo** — ele aparece como `Donor/Desynced`, mas não bloqueia a aplicação. Por isso o mariabackup é o método recomendado (ao contrário de métodos antigos como `rsync`/`mysqldump`, que travavam o doador).

## Quando o SST FALHA — onde procurar

Se o nó fica preso em `Joining` e não chega a `Synced`, investigue nesta ordem:

1. **Log do nó que entra (joiner):**
   ```bash
   sudo journalctl -u mariadb -n 100 --no-pager | grep -iE 'sst|wsrep|error|failed'
   ```
2. **Log do script de SST** (o `mariabackup` roda por fora): procure por `wsrep_sst_mariabackup` no journal dos **dois** nós (doador e joiner).
3. **Causas mais comuns:**
   | Sintoma | Causa provável | Onde conferir |
   |---|---|---|
   | SST aborta na autenticação | `wsrep_sst_auth` errado ou usuário SST sem permissão | `60-galera.cnf` e o usuário `sst_user` (ver [instalação](../Galera/galera-ha-haproxy-keepalived.md)) |
   | SST não conecta / timeout | porta **4444** (SST) bloqueada entre os nós | `ufw`/firewall, `ss -ltnp \| grep 4444` |
   | `mariabackup: command not found` | pacote `mariadb-backup` ausente no doador/joiner | `dpkg -l \| grep mariadb-backup` |
   | SST falha por versão | versões diferentes de MariaDB/Galera entre nós | `SELECT VERSION()` nos dois |
   | SST enche o disco | datadir grande sem espaço no joiner | `df -h /var/lib/mysql` |
4. **Forçar IST em vez de SST** (quando possível): se o nó ficou fora pouco tempo, mantenha o `grastate.dat` (não apague o datadir) — ele tenta IST primeiro.

## Sinais no monitoramento

Durante um state transfer, observe (ver [monitoramento do Galera](../Galera/galera-monitoramento-observabilidade.md)):

```sql
-- No doador e no joiner
SHOW STATUS LIKE 'wsrep_local_state_comment';   -- Donor/Desynced | Joining | Joined | Synced
SHOW STATUS LIKE 'wsrep_cluster_size';
```

## O que NÃO fazer

- **Não apague o datadir de mais de um nó**, nem do nó errado — você pode ficar sem nenhuma cópia boa.
- **Não force SST à toa.** Se o nó ficou fora pouco tempo, o IST é muito mais rápido; apagar o datadir joga fora essa vantagem.
- **Não conclua "travou" cedo demais.** Um SST de uma base grande leva tempo (proporcional aos dados); acompanhe o progresso no log antes de intervir.
- **Não use `rsync`/métodos que travam o doador** em produção — prefira `mariabackup`.

## Veja também

- [Instalação e topologia do cluster](../Galera/galera-ha-haproxy-keepalived.md)
- [Recuperação de queda total](../Galera/galera-recuperacao-queda-total.md)
- [Monitoramento do Galera](../Galera/galera-monitoramento-observabilidade.md)
