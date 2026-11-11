# MariaDB Galera: operação do dia a dia

Tarefas rotineiras de quem cuida do cluster: **reiniciar um nó sem derrubar o serviço** (rolling restart), **fazer backup**, e **adicionar ou remover um nó**. Os procedimentos de rolling restart e backup foram **executados e validados no laboratório** (saídas reais abaixo); o de adicionar/remover nó é o padrão do Galera, com a parte de reentrada comprovada no lab.

> Veja também: [instalação/topologia](galera-ha-haproxy-keepalived.md), [monitoramento](galera-monitoramento-observabilidade.md), [testes de falha](galera-testes-de-falha-failover.md), [queda total](galera-recuperacao-queda-total.md).

## Para quem é este documento

Estagiários e juniores. Estas são as operações que você vai repetir sempre. A regra que vale para todas: **um nó por vez**, sempre esperando o nó voltar a `Synced` antes de mexer no próximo.

---

## 1. Rolling restart (reiniciar sem downtime)

Use quando precisar aplicar uma mudança de configuração, um ajuste de kernel, ou um upgrade controlado — **sem** derrubar a aplicação.

### A regra de ouro

Reinicie **um nó de cada vez**. Enquanto um nó reinicia, os outros atendem. Só passe para o próximo quando o atual voltar a `Synced`.

### Ordem recomendada

Comece pelo nó que **não** é o ativo de escrita (o db-02), depois o db-01.

```bash
# --- Nó db-02 primeiro ---
# No db-02:
sudo systemctl restart mariadb
# Espere voltar a Synced antes de continuar:
sudo mariadb -N -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"   # aguarde 'Synced'
```

No lab, o restart retornou em **~7s** e o nó voltou a `Synced` quase imediatamente, reentrando por **IST** (transferência incremental). Trecho real do journal:

```
WSREP: Shifting JOINED -> SYNCED (TO: 87)
WSREP: Server status change joined -> synced
```

```bash
# --- Depois o db-01 (nó de escrita) ---
# No db-01:
sudo systemctl restart mariadb
sudo mariadb -N -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"   # aguarde 'Synced'
```

O que acontece ao reiniciar o db-01 (observado no lab):

- O **VIP continua no db-01** (o Keepalived olha o HAProxy, que não foi reiniciado).
- Enquanto o MariaDB do db-01 está fora, a **escrita faz failover** para o db-02 (veja [testes de falha](galera-testes-de-falha-failover.md), Teste 2).
- Quando o db-01 volta a `Synced` (~7s + health check), a escrita **retorna** para ele.

### Por que foi tão rápido (IST e não SST)

Como o nó ficou fora por poucos segundos, o `gcache` (configurado com `gcache.size=512M`) ainda tinha todos os writesets que faltavam. Então o Galera mandou só o delta (**IST**), em vez de copiar o banco inteiro (**SST**). No log aparece algo como `Min available from gcache for CC from sst: 55`.

> **Cuidado:** se um nó ficar **muito tempo** parado (mais do que o `gcache` consegue guardar), a reentrada vira **SST** (cópia completa via `mariabackup`), que é demorada e usa o nó doador como fonte. Não deixe um nó parado por horas achando que a volta será instantânea.

### Checklist do rolling restart

- [ ] db-02 reiniciado e de volta em `Synced`
- [ ] `wsrep_cluster_size = 3` antes de mexer no db-01
- [ ] db-01 reiniciado e de volta em `Synced`
- [ ] HAProxy 4× `UP`, VIP no db-01, escrita roteando no db-01

---

## 2. Backup com `mariadb-backup`

`mariadb-backup` faz um backup físico **a quente** (sem travar a aplicação). Em um cluster, rode o backup **a partir do nó de leitura** (db-02), para não pesar no nó de escrita.

### Backup

```bash
# No db-02
sudo mariadb-backup --backup --target-dir=/caminho/do/backup --galera-info
```

Saída real do lab (final do processo):

```
[00] Redo log (from LSN 138215 to 143461) was copied.
[00] completed OK!
```

> **`completed OK!` é a linha que importa.** Se ela não aparecer no fim, o backup **não** é confiável.

Tamanho do backup no lab: **49M** (banco praticamente vazio; em produção será bem maior).

### Arquivos de metadados (nesta versão, 12.3)

Dentro do diretório do backup ficam (nomes **reais** do lab — nas versões antigas chamavam-se `xtrabackup_*`):

```
mariadb_backup_info          <- versão, LSN, horário, comando usado
mariadb_backup_checkpoints   <- tipo (full/incremental) e LSN
```

Conteúdo real do `mariadb_backup_info`:

```
tool_version    = 12.3.3-MariaDB
server_version  = 12.3.3-MariaDB-ubu2604
start_time      = 2026-11-10 23:45:38
end_time        = 2026-11-10 23:45:39
innodb_to_lsn   = 138215
partial         = N
incremental     = N
```

> **`--galera-info`**: adiciona o arquivo `mariadb_backup_galera_info` com a **posição do cluster** (`UUID:seqno`) no momento do backup. É isso que permite reconstruir o ponto exato de replicação ao restaurar um nó Galera. Use sempre esse flag em cluster.

### Preparar (tornar restaurável)

Um backup recém-tirado não está pronto para uso — é preciso "preparar" (aplicar os redo logs):

```bash
sudo mariadb-backup --prepare --target-dir=/caminho/do/backup
#  -> deve terminar com 'completed OK!'
```

### Restaurar (resumo — faça com o MariaDB parado)

```bash
# No nó alvo, com o MariaDB PARADO e o datadir vazio:
sudo systemctl stop mariadb
sudo rm -rf /var/lib/mysql/*
sudo mariadb-backup --copy-back --target-dir=/caminho/do/backup
sudo chown -R mysql:mysql /var/lib/mysql
sudo systemctl start mariadb
```

> Num cluster, muitas vezes a forma mais simples de "restaurar" um nó é **apagar o datadir e deixar o Galera fazer SST** a partir de um nó saudável. O backup com `mariadb-backup` é essencial para o **cluster inteiro** (desastre total) ou para point-in-time, não necessariamente para repor um único nó.

### Boas práticas de backup

- Rode **do db-02** (nó de leitura), com `--galera-info`.
- Agende fora do horário de pico.
- Guarde o backup **fora** das VMs do cluster.
- Teste a restauração periodicamente — backup não testado é esperança, não backup.

---

## 3. Adicionar ou remover um nó

> Esta seção descreve o procedimento padrão do Galera. A parte de **reentrada de um nó** foi comprovada no lab (via restart/rolling); adicionar uma **4ª VM** nova não foi executado por falta de um host extra no laboratório.

### Remover um nó do cluster (permanente)

1. **Pare o MariaDB** no nó a remover — saída graciosa:
   ```bash
   sudo systemctl stop mariadb
   ```
   O cluster recalcula o quórum sozinho (`wsrep_cluster_size` diminui). Isso foi observado no lab: parar um nó com o cluster saudável é rápido e limpo.
2. **Tire o IP do nó** da linha `wsrep_cluster_address` (`gcomm://...`) nos nós que ficam, em `/etc/mysql/mariadb.conf.d/60-galera.cnf`, para ele não ser mais esperado:
   ```
   wsrep_cluster_address = gcomm://192.168.1.21,192.168.1.23
   ```
3. Se o nó removido também era alvo do HAProxy ou do `garbd`, ajuste esses arquivos (`haproxy.cfg`, `/etc/default/garb`).
4. **Desabilite** o serviço no nó removido:
   ```bash
   sudo systemctl disable --now mariadb
   ```

### Adicionar um nó novo

1. Provisione a VM seguindo a [doc de instalação](galera-ha-haproxy-keepalived.md) (mesmo Ubuntu, mesma versão de MariaDB e de `galera-4`).
2. Crie o `60-galera.cnf` com os dados do novo nó (`wsrep_node_name`, `wsrep_node_address`, `bind-address`) e **inclua o novo IP** no `wsrep_cluster_address` de **todos** os nós.
3. Garanta `machine-id` único e o usuário de SST (`sst_user`) com a mesma senha.
4. **Inicie o MariaDB no nó novo** — ele entra sozinho e puxa os dados:
   ```bash
   sudo systemctl start mariadb
   ```
   - Nó novo (datadir vazio) → **SST** (cópia completa via `mariabackup`) a partir de um doador.
   - Nó que já teve dados e voltou rápido → **IST** (incremental). No lab, a reentrada por IST ficou assim:
     ```
     WSREP: Shifting JOINED -> SYNCED (TO: 89)
     ```
5. Confira o crescimento do cluster:
   ```bash
   sudo mariadb -N -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
   ```
6. Inclua o novo nó no HAProxy (`haproxy.cfg`) se ele for atender tráfego.

> **Número de nós e quórum:** prefira manter um número **ímpar de votos**. Com 2 nós de dados, o árbitro (`garbd`) é o 3º voto. Se for para 3 nós de dados, o árbitro deixa de ser necessário (já são ímpares). Com 4 nós de dados, volte a precisar de um árbitro (ou um 5º nó) para desempatar.

---

## Lembretes de manutenção (deste ambiente)

- **Atualizações automáticas estão desligadas** (Passo 1 da instalação). Atualize **um nó por vez**, de forma controlada (rolling), nunca todos juntos.
- **Desligar/ligar o ambiente** tem ordem própria — veja a seção "Desligar e ligar" da [doc de instalação](galera-ha-haproxy-keepalived.md). Desligar os dois nós de dados ao mesmo tempo é o que leva à [queda total](galera-recuperacao-queda-total.md).
- **Versões do `garbd` e do `galera-4`** precisam ser iguais. Confira ao atualizar.
- **`mariabackup` x `mariadb-backup`**: são o mesmo programa; `mariadb-backup` é o nome atual.

## Referência rápida

```bash
# Rolling restart (um nó por vez, esperando Synced)
sudo systemctl restart mariadb
sudo mariadb -N -e "SHOW STATUS LIKE 'wsrep_local_state_comment';"

# Backup (do db-02)
sudo mariadb-backup --backup --target-dir=/bkp/$(date +%F) --galera-info
sudo mariadb-backup --prepare --target-dir=/bkp/$(date +%F)

# Tamanho do cluster
sudo mariadb -N -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
```
