# MariaDB Galera: testes de falha e failover

O que acontece — e o que você deve ver — quando **um** componente do cluster cai, enquanto o resto continua atendendo a aplicação. São os testes que provam que a alta disponibilidade funciona de verdade.

> Procedimentos **executados e validados em laboratório**. Cada saída abaixo é real. Topologia e instalação: [galera-ha-haproxy-keepalived.md](galera-ha-haproxy-keepalived.md). Para o cenário de **queda total**, veja [galera-recuperacao-queda-total.md](galera-recuperacao-queda-total.md).

## Para quem é este documento

Estagiários e juniores. A meta é você entender **a cadeia de reações** a cada falha: quem detecta, em quanto tempo, para onde o tráfego vai, e como voltar ao normal. Faça cada teste com o cluster saudável e **sempre reverta** antes do próximo.

## Antes de começar: o "estado saudável"

Todo teste parte daqui e volta para cá. Decore esta foto:

```bash
# db-01
sudo mariadb -N -e "SHOW STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status');"
#  -> wsrep_cluster_size = 3 / wsrep_cluster_status = Primary
ip -br a show enp0s8 | grep 192.168.1.100        # VIP só no db-01
curl -s 'http://127.0.0.1:8404/stats;csv' | cut -d, -f1,2,18 | column -s, -t | grep -E 'mariadb_(write|read)'
#  -> os 4 backends UP
```

### Como a aplicação enxerga o cluster (recapitulando os dois caminhos)

- **Porta 3306 (escrita):** o HAProxy manda **sempre para o db-01**; o db-02 é `backup` e só recebe se o db-01 cair.
- **Porta 3307 (leitura):** o HAProxy **alterna** entre db-01 e db-02.

Isso foi confirmado no lab com uma sonda que só roda `SELECT @@hostname` pelo VIP:

```
# 6 conexões na 3306 (escrita):   db-01, db-01, db-01, db-01, db-01, db-01
# 6 conexões na 3307 (leitura):   db-01, db-02, db-01, db-02, db-01, db-02
```

### Como o HAProxy decide que um nó caiu

Cada backend é checado pela porta **9200** (o *health check*), que responde `200` só quando o nó está `Synced`. A configuração é `inter 2s fall 3 rise 2`:

- **`fall 3`** → precisa de **3 checagens falhas seguidas** (≈ 6 segundos) para marcar o nó como `DOWN`.
- **`rise 2`** → precisa de **2 checagens boas** (≈ 4 segundos) para voltar a `UP`.

Esse atraso de detecção é esperado e aparece nos testes abaixo como `UP 2/3` (nó ainda UP, já com 2 de 3 falhas acumuladas).

---

## Teste 1 — queda de um nó de dados (db-02)

**Hipótese:** o cluster continua `Primary` com 2 membros + árbitro; o HAProxy tira o db-02 de rotação; a escrita segue no db-01.

### Ação

```bash
# No db-02
sudo systemctl stop mariadb
```

No lab a parada foi **graciosa e rápida (1s)** — diferente da queda total, aqui o nó consegue avisar o grupo antes de sair (ele ainda enxerga o db-01 e o árbitro).

### O que observar (no db-01)

```bash
sudo mariadb -N -e "SHOW STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status','wsrep_local_state_comment');"
```

```
wsrep_local_state_comment   Synced
wsrep_cluster_size          2          <- caiu de 3 para 2
wsrep_cluster_status        Primary    <- continua Primary (tem quórum)
```

Health check do nó caído e stats do HAProxy:

```bash
# No db-02 (está parado) -> o health check responde 503
curl -s -o /dev/null -w 'http=%{http_code}\n' http://127.0.0.1:9200     # http=503

# No db-01 -> logo após a queda, pode pegar o HAProxy "no meio da detecção"
curl -s 'http://127.0.0.1:8404/stats;csv' | cut -d, -f1,2,18 | column -s, -t | grep -E 'mariadb_(write|read)'
```

Saída real capturada **durante** a janela de detecção (repare no `UP 2/3`):

```
mariadb_write  mariadb-db-01  UP
mariadb_write  mariadb-db-02  UP 2/3     <- 2 de 3 falhas; vira DOWN em ~6s
mariadb_read   mariadb-db-01  UP
mariadb_read   mariadb-db-02  UP 2/3
```

A escrita no db-01 nunca parou:

```bash
sudo mariadb -e "INSERT INTO _ha_check.t(src) VALUES('durante-queda-db02');"   # ok
```

### Reversão

```bash
# No db-02
sudo systemctl start mariadb
```

Volta para `wsrep_cluster_size = 3` e os 4 backends do HAProxy `UP`.

> **Aprendizado:** perder um nó de dados **não derruba** o serviço. A aplicação pode sentir, por alguns segundos, leituras na 3307 caindo no nó que está morrendo, até o HAProxy marcá-lo `DOWN` (os ~6s do `fall 3`).

---

## Teste 2 — failover de escrita (queda do nó de escrita ativo, db-01)

**Hipótese:** parando o **MariaDB** do db-01 (mas deixando HAProxy e Keepalived de pé), o VIP **continua no db-01**, e o HAProxy passa a escrita para o db-02 (o `backup`).

> Repare na diferença entre "cair o banco" e "cair o HAProxy". Aqui só o banco cai; o VIP não se mexe (quem move o VIP é o Keepalived, e ele olha o HAProxy, não o MariaDB). Mover o VIP é o Teste 3.

### Ação

```bash
# No db-01
sudo systemctl stop mariadb
# espere ~9s para o HAProxy marcar o db-01 DOWN (fall 3 x inter 2s)
```

### O que observar

```bash
# O VIP continua no db-01?
# No db-01:
ip -br a show enp0s8 | grep 192.168.1.100      # ainda aparece aqui -> VIP ficou no db-01
```

```bash
# Cluster visto do db-02 (que virou o nó ativo)
# No db-02:
sudo mariadb -N -e "SHOW STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status');"
#  -> wsrep_cluster_size = 2 / Primary

# HAProxy (o do db-01 segue rodando e servindo o VIP)
curl -s 'http://127.0.0.1:8404/stats;csv' | cut -d, -f1,2,18 | column -s, -t | grep mariadb_write
```

Saída real:

```
mariadb_write  mariadb-db-01  DOWN
mariadb_write  mariadb-db-02  UP 2/3
```

A escrita pelo VIP agora cai no **db-02**:

```
3306 -> mariadb-db-02        # era db-01 com tudo saudável
```

### Reversão

```bash
# No db-01
sudo systemctl start mariadb
```

Quando o db-01 volta a `Synced` e passa no health check (`rise 2`, ~4s), a escrita **volta para o db-01** automaticamente:

```
escrita 3306 -> mariadb-db-01
```

> **Aprendizado:** a escrita tem failover automático **sem mover o VIP** — o próprio HAProxy redireciona para o `backup`. O cliente só precisa reconectar; conexões abertas no nó que caiu são encerradas (`on-marked-down shutdown-sessions`).

---

## Teste 3 — migração do VIP (queda do HAProxy do db-01)

**Hipótese:** parando o **HAProxy** do db-01, o Keepalived detecta (via `pidof haproxy`), a prioridade do db-01 cai de 110 para 90 (peso `-20`), e o VIP migra para o db-02 (prioridade 100).

### Ação

```bash
# No db-01
sudo systemctl stop haproxy
# espere ~7s (chk interval 2s, fall 2, + anúncio VRRP)
```

### O que observar

```bash
# VIP saiu do db-01?
# db-01:
ip -br a show enp0s8 | grep 192.168.1.100 || echo '(sem VIP)'      # (sem VIP)
# db-02:
ip -br a show enp0s8 | grep 192.168.1.100                          # 192.168.1.100/24  <- migrou!
```

Transição de estado no Keepalived do db-02 (journal real):

```bash
sudo journalctl -u keepalived -n 15 --no-pager -o cat | grep -iE 'MASTER|BACKUP'
```

```
(VI_MARIADB) Entering MASTER STATE          <- db-02 assumiu o VIP
```

A aplicação continua acessível pelo VIP (agora servida pelo HAProxy do db-02), e a escrita segue roteada para o db-01 (que está vivo):

```
escrita 3306 -> mariadb-db-01
stats 8404  -> 200
```

### Reversão

```bash
# No db-01
sudo systemctl start haproxy
# espere ~6s
```

Com o HAProxy de volta, a prioridade do db-01 volta a 110 e o VIP é **preemptado** de volta para o db-01. O journal do db-02 mostra a devolução:

```
(VI_MARIADB) Master received advert from 192.168.1.21 with higher priority 110, ours 100
(VI_MARIADB) Entering BACKUP STATE
```

Confirme:

```
VIP db-01: 192.168.1.100/24
VIP db-02: (sem VIP)
```

> **Aprendizado:** o VIP sempre segue o **HAProxy saudável de maior prioridade**. Como não há `nopreempt` na configuração, o db-01 **retoma** o VIP ao voltar. Em produção, avalie se essa "sacudida" extra do VIP vale a pena, ou se `nopreempt` (deixar no db-02 até ele falhar) seria mais tranquilo.

---

## Teste 4 — queda do árbitro (db-03)

**Hipótese:** com os dois nós de dados vivos, perder o árbitro **não afeta nada** — o cluster segue `Primary` com `size=2`.

### Ação

```bash
# No db-03
sudo systemctl stop garb
```

### O que observar (no db-01)

```bash
sudo mariadb -N -e "SHOW STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status','wsrep_local_state_comment');"
```

```
wsrep_local_state_comment   Synced
wsrep_cluster_size          2
wsrep_cluster_status        Primary
```

Escrita continua normal:

```bash
sudo mariadb -e "INSERT INTO _ha_check.t(src) VALUES('sem-arbitro');"   # ok
```

### Reversão

```bash
# No db-03
sudo systemctl restart garb
#  -> wsrep_cluster_size volta a 3
```

> **Aprendizado:** enquanto os **dois nós de dados se enxergam**, o árbitro é dispensável. Ele só faz diferença quando **um nó de dados cai** — aí ele é o terceiro voto que mantém o quórum. É exatamente o que o próximo documento demonstra: tire o árbitro **e** derrube a comunicação entre os nós, e o cluster trava por falta de maioria. Veja [galera-perda-de-quorum-recuperacao.md](galera-perda-de-quorum-recuperacao.md).

---

## Tabela-resumo: quem cai, quem assume

| Falha | `cluster_size` | `cluster_status` | VIP | Escrita (3306) | Detecção |
|---|---|---|---|---|---|
| **1 nó de dados (db-02)** | 3 → 2 | Primary | fica no db-01 | segue no db-01 | HAProxy ~6s (`fall 3`) |
| **MariaDB do db-01** | 3 → 2 | Primary | fica no db-01 | **failover p/ db-02** | HAProxy ~6s |
| **HAProxy do db-01** | 3 (inalterado) | Primary | **migra p/ db-02** | segue no db-01 | Keepalived ~4s (`fall 2`) |
| **Árbitro (db-03)** | 3 → 2 | Primary | fica no db-01 | segue no db-01 | — (sem impacto) |

## Boas práticas ao testar

- **Um teste por vez**, sempre voltando ao estado saudável antes do próximo.
- **Valide pelo HAProxy**, não só pelo `curl` local do health check — o que importa é como o balanceador enxerga o nó (`stats;csv`).
- **Dê tempo à detecção.** Se você olhar rápido demais, vê `UP 2/3` em vez de `DOWN`; não é bug, é o `fall`/`rise`.
- **Cuidado com `leastconn` em leitura:** conexões rápidas e sequenciais podem cair sempre no mesmo nó. Para ver o balanceamento, use conexões simultâneas (ex.: várias com `SLEEP`).

## Veja também

- [galera-ha-haproxy-keepalived.md](galera-ha-haproxy-keepalived.md) — instalação e topologia.
- [galera-recuperacao-queda-total.md](galera-recuperacao-queda-total.md) — quando **tudo** cai.
- [galera-perda-de-quorum-recuperacao.md](galera-perda-de-quorum-recuperacao.md) — split-brain e perda de maioria.
