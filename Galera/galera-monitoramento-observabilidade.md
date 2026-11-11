# MariaDB Galera: monitoramento e observabilidade

O que olhar para saber se o cluster está **saudável** — e, principalmente, para perceber um problema **antes** de virar incidente. Todas as variáveis e saídas abaixo foram coletadas do laboratório real.

> Veja também: [instalação/topologia](galera-ha-haproxy-keepalived.md), [testes de falha](galera-testes-de-falha-failover.md), [queda total](galera-recuperacao-queda-total.md), [perda de quórum](galera-perda-de-quorum-recuperacao.md).

## Para quem é este documento

Estagiários e juniores de plantão. A meta: você bater o olho em 3 ou 4 números e saber dizer "está tudo bem" ou "tem algo errado aqui" — sem decorar.

## As 3 perguntas que o monitoramento responde

1. **Este nó faz parte de um cluster com quórum?** (está `Primary` e `Synced`?)
2. **Este nó está acompanhando o ritmo?** (tem fila/*flow control* se acumulando?)
3. **O balanceador está roteando certo?** (HAProxy enxerga os nós como deve?)

## 1. Saúde do nó e do cluster (as variáveis `wsrep_*`)

Comando base (roda via `sudo`, usando o `unix_socket` do root, sem senha):

```bash
sudo mariadb -e "SHOW GLOBAL STATUS LIKE 'wsrep_%';"
```

As variáveis que **de fato importam** (valores reais do lab, cluster saudável):

| Variável | Valor saudável (lab) | O que significa | Alerta se… |
|---|---|---|---|
| `wsrep_cluster_status` | `Primary` | O nó está no componente com quórum | **≠ Primary** → nó read-only (sem maioria) |
| `wsrep_local_state_comment` | `Synced` | O nó está sincronizado e atendendo | `Donor/Desynced`, `Joining`, `Initialized` por muito tempo |
| `wsrep_ready` | `ON` | O nó aceita queries de aplicação | `OFF` → recusa leitura/escrita (erro 1047) |
| `wsrep_connected` | `ON` | Conectado à rede do cluster | `OFF` |
| `wsrep_cluster_size` | `3` | Membros no cluster (árbitro conta) | caiu sem motivo conhecido |
| `wsrep_local_state` | `4` | Código do estado (4 = Synced) | ≠ 4 |
| `wsrep_cluster_conf_id` | `13` | Nº da "configuração" do cluster; muda a cada entrada/saída | sobe rápido = membros entrando/saindo demais (instabilidade) |
| `wsrep_local_index` | `2` | Índice deste nó na lista de membros | — |
| `wsrep_evs_state` | `OPERATIONAL` | Estado do protocolo de comunicação (EVS) | ≠ OPERATIONAL |
| `wsrep_evs_delayed` | *(vazio)* | Lista de nós com comunicação atrasada | **não vazio** → rede ruim entre nós |
| `wsrep_provider_version` | `26.4.27(rf032a460)` | Versão do Galera (tem que bater com o `garbd`) | divergência entre nós |

> **Os 3 que respondem "está vivo?":** `wsrep_cluster_status = Primary`, `wsrep_local_state_comment = Synced`, `wsrep_ready = ON`. Se os três estão certos, o nó está atendendo. **`SELECT 1` não serve** — ele responde até em nó `non-Primary` (veja [perda de quórum](galera-perda-de-quorum-recuperacao.md)).

Comando enxuto para o plantão:

```bash
sudo mariadb -N -e "SHOW GLOBAL STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status','wsrep_local_state_comment','wsrep_ready','wsrep_connected','wsrep_evs_state','wsrep_evs_delayed');"
```

## 2. O nó está acompanhando o ritmo? (fila e *flow control*)

Em Galera, se um nó fica lento, ele pede ao cluster para **segurar as escritas** (*flow control*) até ele se recuperar. Isso protege a consistência, mas **segura o cluster inteiro**. Por isso é um sinal importante.

| Variável | Valor saudável (lab) | O que significa | Alerta se… |
|---|---|---|---|
| `wsrep_flow_control_paused` | `0` | Fração do tempo (0 a 1) em que o cluster ficou pausado por este nó | **> 0,1** (nó segurando o cluster) |
| `wsrep_flow_control_sent` | `0` | Quantas vezes este nó pediu pausa | subindo = nó não acompanha |
| `wsrep_local_recv_queue_avg` | `0.15` | Fila média de writesets a aplicar | **> 1** de forma sustentada |
| `wsrep_local_send_queue_avg` | `0` | Fila média a enviar | **> 1** (rede saturada) |
| `wsrep_cert_deps_distance` | `1.125` | Paralelismo possível ao aplicar | ajuda a dimensionar `wsrep_slave_threads` |
| `wsrep_local_cert_failures` | `0` | Conflitos de certificação (deadlocks distribuídos) | subindo = app escrevendo nos mesmos registros em nós diferentes |
| `wsrep_local_bf_aborts` | `0` | Transações locais abortadas por conflito | subindo = idem acima |

```bash
sudo mariadb -N -e "SHOW GLOBAL STATUS WHERE Variable_name IN ('wsrep_flow_control_paused','wsrep_local_recv_queue_avg','wsrep_local_send_queue_avg','wsrep_cert_deps_distance','wsrep_local_cert_failures','wsrep_local_bf_aborts');"
```

> **Regra de bolso:** `wsrep_flow_control_paused` perto de `0` e filas perto de `0` = cluster folgado. Se o `flow_control_paused` passa de ~0,1, algum nó está atrasando todo mundo (disco lento, nó sobrecarregado, rede ruim).

## 3. O balanceador está roteando certo? (HAProxy)

O HAProxy tem uma página de estatísticas na porta **8404**. Para automação, use o CSV:

```bash
curl -s 'http://127.0.0.1:8404/stats;csv' | cut -d, -f1,2,18,19,20,21,37,38 | column -s, -t | grep -E 'mariadb_(write|read).*(db-01|db-02)'
```

Saída real do lab (saudável):

```
pxname         svname         status  weight  act  bck  check_status  check_code
mariadb_write  mariadb-db-01  UP      1       1    0    L7OK          200
mariadb_write  mariadb-db-02  UP      1       0    1    L7OK          200    <- backup (bck=1)
mariadb_read   mariadb-db-01  UP      1       1    0    L7OK          200
mariadb_read   mariadb-db-02  UP      1       1    0    L7OK          200
```

Como ler:

- **`status = UP`** e **`check_status = L7OK` / `check_code = 200`**: o nó passou no health check. `L7TOUT` = timeout do check; `DOWN` = fora de rotação.
- **`act` / `bck`**: no serviço de **escrita**, o db-02 é `bck=1` (backup, só entra se o db-01 cair). No de **leitura**, os dois são `act=1` (ativos, balanceados). Isso confirma a topologia "escrita num nó, leitura nos dois".
- **`UP 2/3`** (aparece como status): o nó ainda está UP, mas já acumulou 2 de 3 checagens falhas — vai virar `DOWN` em instantes (`fall 3` × `inter 2s` ≈ 6s).

Pela interface web (navegador ou `curl`): `http://<VIP>:8404/stats`.

## 4. O health check (porta 9200) por dentro

O HAProxy pergunta a cada nó, na porta 9200, "você está `Synced`?". Resposta real quando saudável:

```bash
curl -si http://127.0.0.1:9200
```

```
HTTP/1.1 200 OK
Content-Type: text/plain
Connection: close
Content-Length: 14

Galera Synced
```

A lógica: o script lê `wsrep_local_state`; se for **`4` (Synced)** responde `200 OK / Galera Synced`, senão `503 / Galera Not Synced`. É por isso que um nó `Donor`, `Joining` ou `non-Primary` sai da rotação automaticamente — ele não está no estado `4`.

> **Importante:** teste o health check **pelo HAProxy** (`stats;csv`), não só com `curl` local. Um `curl` que funciona não garante que o HAProxy esteja enxergando o nó (timeout, firewall, etc.).

## 5. Observar continuamente (plantão)

Painel "ao vivo" no terminal (atualiza a cada 2s):

```bash
watch -n2 "sudo mariadb -N -e \"SHOW GLOBAL STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status','wsrep_local_state_comment','wsrep_ready','wsrep_flow_control_paused','wsrep_local_recv_queue_avg')\""
```

VIP e serviços:

```bash
watch -n2 "ip -br a show enp0s8 | grep 192.168.1.100; systemctl is-active mariadb haproxy keepalived garb 2>/dev/null"
```

Onde ficam os logs (lembrete: **não existe** `/var/log/mysql/error.log` nesta instalação):

```bash
sudo journalctl -u mariadb   -n 50 --no-pager
sudo journalctl -u haproxy   -n 50 --no-pager
sudo journalctl -u keepalived -n 50 --no-pager
sudo journalctl -u garb      -n 50 --no-pager   # no db-03
```

## 6. O que vale a pena alertar (resumo para quem for montar alarme)

| Condição | Gravidade | Por quê |
|---|---|---|
| `wsrep_cluster_status != Primary` | **crítico** | nó read-only, perdeu quórum |
| `wsrep_ready = OFF` | **crítico** | recusa queries da aplicação |
| `wsrep_cluster_size < 3` (sustentado) | alto | perdeu um membro; HA reduzida |
| HAProxy backend `DOWN` | alto | um nó fora de rotação |
| `wsrep_flow_control_paused > 0.1` | alto | um nó segurando o cluster |
| `wsrep_local_recv_queue_avg > 1` (sustentado) | médio | nó atrasando |
| `wsrep_evs_delayed` não vazio | médio | rede ruim entre nós |
| `wsrep_cluster_conf_id` subindo rápido | médio | membros entrando/saindo (instabilidade) |
| `wsrep_local_cert_failures` / `bf_aborts` subindo | médio | app em conflito de escrita entre nós |
| VIP ausente nos dois nós | **crítico** | aplicação sem ponto de entrada |

## Referência rápida (cola de plantão)

```bash
# "Está tudo bem?" em um comando
sudo mariadb -N -e "SHOW GLOBAL STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status','wsrep_local_state_comment','wsrep_ready');"

# Fila/flow control
sudo mariadb -N -e "SHOW GLOBAL STATUS LIKE 'wsrep_flow_control_paused';"

# Balanceador
curl -s 'http://127.0.0.1:8404/stats;csv' | cut -d, -f1,2,18,37,38 | column -s, -t | grep -E 'mariadb_(write|read)'

# VIP
ip -br a show enp0s8 | grep 192.168.1.100
```

Estado "tudo verde" (confirmado no lab): `size=3`, `Primary`, `Synced`, `ready=ON`, `flow_control_paused=0`, filas ~0, HAProxy 4× `UP/L7OK`, VIP no db-01.
