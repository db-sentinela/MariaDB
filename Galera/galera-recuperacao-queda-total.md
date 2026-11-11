# MariaDB Galera: recuperação de queda total (cluster todo parado)

Como trazer o cluster de volta quando **todos os nós pararam** e nenhum sobe sozinho. Este é o cenário mais assustador para quem está começando, mas o procedimento é mecânico quando você entende a regra de ouro.

> Procedimento **executado e validado em laboratório** (os nós, IPs e saídas abaixo são reais). Topologia e instalação: veja [galera-ha-haproxy-keepalived.md](galera-ha-haproxy-keepalived.md).

## Para quem é este documento

Estagiários e juniores que vão dar plantão no cluster. A ideia é que você consiga executar a recuperação **lendo e entendendo cada passo**, não decorando.

## O conceito em uma frase

O Galera só aceita trabalhar quando a **maioria** (quórum) dos membros está no ar. Se o cluster inteiro caiu, não existe maioria para "puxar" ninguém — então **um** nó precisa ser escolhido à mão para recomeçar o cluster (dar o *bootstrap*). A única pergunta difícil é: **qual nó?**

A resposta: **o nó que tinha os dados mais recentes** (a maior posição de replicação, o `seqno`). Se você escolher um nó atrasado, perde as transações que só existiam no nó mais avançado.

## Mapa dos nós (lembrete)

| Host | IP | Papel |
|---|---|---|
| mariadb-db-01 | 192.168.1.21 | nó de dados + HAProxy + Keepalived MASTER |
| mariadb-db-02 | 192.168.1.23 | nó de dados + HAProxy + Keepalived BACKUP |
| mariadb-db-03 | 192.168.1.24 | árbitro (`garb`) |

## Passo 0: reconhecer o problema

Sintomas de uma queda total com *shutdown* travado (o que aconteceu de verdade no lab):

```bash
# Em cada nó de dados
systemctl is-active mariadb
```

No lab apareceu `deactivating` (nem ativo, nem parado) nos dois nós — o `mariadbd` recebeu o pedido de parada mas **não consegue encerrar**, porque tenta avisar o grupo antes de sair e não acha ninguém:

```bash
sudo journalctl -u mariadb -n 15 --no-pager -o cat
```

Saída real (repetindo sem parar):

```
[Note] WSREP: Failed to establish connection: Connection refused
[Note] WSREP: Failed to establish connection: Connection refused
...
```

E o árbitro, sem nenhum nó para conversar, aparece como `failed`:

```bash
# No db-03
systemctl is-active garb      # -> failed
```

> **Leitura do sintoma:** `deactivating` + laço de `Connection refused` = cada nó está isolado, tentando sair do cluster de forma educada, mas não há cluster. Ninguém vai sair desse estado sozinho.

## Passo 1: parar de verdade o MariaDB travado (nos dois nós de dados)

Enquanto o processo está pendurado, nada feito. Primeiro mate o processo preso e confirme que não sobrou nada.

```bash
# Em 192.168.1.21 e 192.168.1.23
sudo systemctl kill -s SIGKILL mariadb
sudo systemctl stop mariadb            # garante que o systemd não vai reiniciar sozinho
sudo systemctl reset-failed mariadb
systemctl is-active mariadb            # esperado: inactive
pgrep -a mariadbd                      # esperado: nada
```

Resultado esperado em cada nó:

```
is-active: inactive
(nenhum mariadbd)
```

### Armadilhas reais deste passo

- **O systemd tenta reiniciar sozinho.** Logo após o `kill`, o serviço pode ir para `activating` de novo (política de restart). Por isso o `systemctl stop` em seguida é importante: ele marca a parada como intencional.
- **O `stop` pode demorar.** No lab ele levou mais de um minuto, porque o systemd estava esperando um *job* de start travado terminar. Isso é normal nesse estado. Se precisar destravar na marra: `sudo pkill -9 -x mariadbd`.

## Passo 2: descobrir qual nó está mais avançado

Nunca escolha o nó de bootstrap "no chute". Pergunte ao próprio Galera qual posição cada nó recuperou:

```bash
# Em cada nó de dados
sudo -u mysql mariadbd --wsrep-recover 2>&1 | grep -i 'Recovered position'
```

Saída real do lab (os dois nós):

```
[Note] WSREP: Recovered position: e516ffd2-d88c-11f1-a1d3-2b2c2ce0dda8:56
```

Compare o número depois dos dois-pontos (o `seqno`):

- db-01 → `...:56`
- db-02 → `...:56`

**Os dois estão em `56` — empatados.** Quando empata, qualquer um serve; por convenção escolhemos o **db-01** (o MASTER do Keepalived). Se estivessem diferentes, o nó do **maior** `seqno` vence.

> Confira também o `grastate.dat`. Depois de uma queda suja ele mostra `seqno: -1` e `safe_to_bootstrap: 0` — isto é, "não sei exatamente onde parei, e não fui eu o último a sair de forma limpa". Por isso rodamos o `--wsrep-recover` acima.

```bash
sudo cat /var/lib/mysql/grastate.dat
```

```
# GALERA saved state
version: 2.1
uuid:    e516ffd2-d88c-11f1-a1d3-2b2c2ce0dda8
seqno:   -1
safe_to_bootstrap: 0
```

## Passo 3: dar o bootstrap no nó escolhido (db-01)

Como provamos que o db-01 está na posição mais alta (empatado no topo), marcamos ele como seguro para bootstrap e iniciamos o cluster a partir dele.

```bash
# SOMENTE no db-01
sudo sed -i 's/^safe_to_bootstrap: 0/safe_to_bootstrap: 1/' /var/lib/mysql/grastate.dat
sudo grep safe_to_bootstrap /var/lib/mysql/grastate.dat     # deve mostrar: safe_to_bootstrap: 1
sudo galera_new_cluster
```

Valide:

```bash
sudo mariadb -N -e "SHOW STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status','wsrep_local_state_comment','wsrep_ready','wsrep_connected');"
```

Resultado real:

```
wsrep_local_state_comment   Synced
wsrep_cluster_size          1
wsrep_cluster_status        Primary
wsrep_connected             ON
wsrep_ready                 ON
```

> **Por que editar o `safe_to_bootstrap`?** Com `seqno` desconhecido e `safe_to_bootstrap: 0`, o `galera_new_cluster` se recusa a subir ("It may not be safe to bootstrap..."), justamente para você não ressuscitar o cluster a partir de um nó atrasado. Você só força `1` **depois** de confirmar, com o `--wsrep-recover`, que esse nó é o mais avançado.

## Passo 4: trazer o segundo nó de dados (db-02)

Com o db-01 de pé como `Primary`, o db-02 entra sozinho ao iniciar o serviço normalmente:

```bash
# No db-02
sudo systemctl start mariadb
```

Confira o tipo de sincronização no log dele:

```bash
sudo journalctl -u mariadb -n 20 --no-pager -o cat | grep -iE 'IST|SST|synced'
```

Saída real (entrou por **IST**, transferência incremental — rápida, porque estava quase igual ao db-01):

```
WSREP: Member 1.0 (mariadb-db-02) synced with group.
WSREP: Shifting JOINED -> SYNCED (TO: 58)
WSREP: Server status change joined -> synced
```

Visto do db-01, agora são dois:

```
wsrep_cluster_size      2
wsrep_cluster_status    Primary
```

> **IST x SST:** o db-02 estava na mesma posição, então o Galera mandou só os poucos eventos que faltavam (**IST** = Incremental State Transfer). Se o nó estivesse muito atrasado ou com dados sujos, ele faria um **SST** (cópia completa via `mariabackup`), que é bem mais demorado.

## Passo 5: religar o árbitro (db-03)

```bash
# No db-03
sudo systemctl restart garb
systemctl is-active garb       # esperado: active
```

Saída real do journal do `garb` ao entrar:

```
INFO: Shifting JOINED -> SYNCED (TO: 59)
INFO: Member 0.0 (mariadb-db-01) synced with group.
```

Agora o cluster conta 3 membros (o árbitro conta como membro, mesmo sem guardar dados):

```
wsrep_cluster_size      3
wsrep_cluster_status    Primary
```

## Passo 6: validação final (a parte que ninguém pode pular)

```bash
# db-01 e db-02
sudo mariadb -N -e "SHOW STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status','wsrep_local_state_comment','wsrep_ready');"
systemctl is-active mariadb haproxy keepalived galera-check.socket
ip -br a show enp0s8 | grep 192.168.1.100          # VIP deve estar só no db-01
curl -s 'http://127.0.0.1:8404/stats;csv' | cut -d, -f1,2,18,37 | column -s, -t | grep -E 'mariadb_(write|read)'
```

Checklist de "voltou de verdade" (tudo confirmado no lab):

- [x] `wsrep_cluster_size = 3`, `wsrep_cluster_status = Primary`, `Synced`, `wsrep_ready = ON`
- [x] `mariadb`, `haproxy`, `keepalived`, `galera-check.socket` → `active` nos dois nós
- [x] `garb` → `active` no db-03
- [x] VIP `192.168.1.100` presente **só no db-01**
- [x] HAProxy com os 4 backends (`mariadb-db-01`/`db-02` em `write` e `read`) `UP` / `L7OK`

Teste de replicação de ponta a ponta (escreve num nó, lê no outro):

```bash
# db-01: escreve
sudo mariadb -e "CREATE DATABASE IF NOT EXISTS _chk; CREATE TABLE IF NOT EXISTS _chk.t(id INT PRIMARY KEY AUTO_INCREMENT, src VARCHAR(30)); INSERT INTO _chk.t(src) VALUES('db-01');"
# db-02: lê (a linha tem que aparecer) e depois limpa
sudo mariadb -N -e "SELECT * FROM _chk.t;"
sudo mariadb -e "DROP DATABASE _chk;"
```

## Resumo visual da recuperação

| Passo | Comando-chave | Onde | Resultado |
|---|---|---|---|
| 1 | `systemctl kill -s SIGKILL` + `stop` + `reset-failed` | db-01, db-02 | `inactive`, sem processo |
| 2 | `mariadbd --wsrep-recover` | db-01, db-02 | descobre o `seqno` de cada um |
| 3 | `safe_to_bootstrap: 1` + `galera_new_cluster` | **db-01** (mais avançado) | `size=1`, `Primary` |
| 4 | `systemctl start mariadb` | db-02 | entra por IST, `size=2` |
| 5 | `systemctl restart garb` | db-03 | `size=3` |
| 6 | validação completa | todos | cluster saudável |

## O que NÃO fazer (erros clássicos)

- **Não dê `galera_new_cluster` em dois nós.** Isso cria dois clusters separados (split-brain). O bootstrap é em **um** nó só.
- **Não escolha o nó de bootstrap no chute.** Sempre rode o `--wsrep-recover` antes. Escolher o nó atrasado = perder dados.
- **Não force `safe_to_bootstrap: 1` sem conferir o `seqno`.** O `0` está lá para te proteger.
- **Não desligue a VM "na tomada" (power off do VirtualBox).** Sempre `shutdown` normal — é o que evita cair neste estado sujo.
- **Não confie em "o serviço está rodando".** Rodando ≠ `Synced` ≠ `Primary`. Sempre olhe o `wsrep_cluster_status` e o `wsrep_ready`.

## Comandos de diagnóstico rápido

```bash
# Estado resumido do cluster
sudo mariadb -e "SHOW STATUS LIKE 'wsrep_%';" | grep -E 'cluster_size|cluster_status|local_state_comment|ready|connected'

# Posição recuperada de um nó
sudo -u mysql mariadbd --wsrep-recover 2>&1 | grep -i 'Recovered position'

# Estado salvo em disco
sudo cat /var/lib/mysql/grastate.dat

# Logs
sudo journalctl -u mariadb -n 50 --no-pager
sudo journalctl -u garb -n 50 --no-pager
```

## Veja também

- [galera-ha-haproxy-keepalived.md](galera-ha-haproxy-keepalived.md) — instalação, topologia e desligar/ligar ordenado.
- [galera-testes-de-falha-failover.md](galera-testes-de-falha-failover.md) — o que acontece quando **um** componente cai (e o cluster continua de pé).
- [galera-perda-de-quorum-recuperacao.md](galera-perda-de-quorum-recuperacao.md) — perda de quórum e proteção contra split-brain.
