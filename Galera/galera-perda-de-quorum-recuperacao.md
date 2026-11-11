# MariaDB Galera: perda de quórum e proteção contra split-brain

Por que, às vezes, o cluster **se recusa a trabalhar de propósito** — e por que isso é um recurso de segurança, não um defeito. Este documento mostra, com uma falha provocada em laboratório, o que é quórum, o que é *split-brain*, e como o Galera se protege.

> Procedimento **executado e validado em laboratório** (partição de rede real provocada com `iptables`). Topologia e instalação: [galera-ha-haproxy-keepalived.md](galera-ha-haproxy-keepalived.md).

## Para quem é este documento

Estagiários e juniores. Este é o conceito que mais confunde no começo: **"o banco está rodando, mas não deixa eu escrever"**. Depois daqui você vai entender exatamente por quê.

## Os conceitos em três frases

- **Quórum** = maioria dos membros do cluster conseguindo se enxergar. Com 3 membros, a maioria é 2.
- **Split-brain** = o cluster se parte em dois pedaços que não se falam, e os dois acham que mandam. Se os dois aceitarem escrita, os dados divergem e não dá mais para juntar.
- **Proteção do Galera:** um pedaço que **não tem maioria** vira `non-Primary` e **para de atender** (recusa leitura e escrita). Melhor ficar indisponível do que corromper os dados.

É por isso que existe o **árbitro**: ele é o terceiro voto que garante que, ao perder um nó de dados, o pedaço que sobra ainda seja maioria (2 de 3).

## O experimento

Para **ver** a perda de quórum, montamos o pior caso de propósito:

1. Paramos o árbitro (ficam só 2 nós de dados — número par, o cenário frágil).
2. Cortamos a comunicação do Galera (porta 4567) **entre** db-01 e db-02, simulando um cabo de rede rompido.

Com 2 membros e a comunicação cortada, cada nó enxerga só a si mesmo: **1 de 2 não é maioria**. Resultado esperado: os **dois** viram `non-Primary`.

> **Por que isto é diferente de "parar um nó"?** Quando você *para* um nó (Teste 4 do [failover](galera-testes-de-falha-failover.md)), ele avisa o grupo que está saindo, e o quórum é recalculado de forma limpa. Numa *partição*, ninguém avisa nada — cada lado só percebe que o outro sumiu. É o caso perigoso, e é o que provoca o `non-Primary`.

## Passo 1: montar o cenário frágil (parar o árbitro)

```bash
# No db-03
sudo systemctl stop garb
```

Confira que o cluster ficou com 2 membros, ainda saudável:

```bash
# db-01
sudo mariadb -N -e "SHOW STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status');"
#  -> wsrep_cluster_size = 2 / Primary
```

## Passo 2: provocar a partição de rede (no db-02)

Bloqueamos a porta **4567/tcp** (a porta de *group communication* do Galera) entre o db-02 e o db-01, nos dois sentidos:

```bash
# No db-02 — corta a conversa do Galera com o db-01
sudo iptables -I INPUT  1 -s 192.168.1.21 -p tcp --dport 4567 -j DROP
sudo iptables -I OUTPUT 1 -d 192.168.1.21 -p tcp --dport 4567 -j DROP
sudo iptables -I INPUT  2 -s 192.168.1.21 -p tcp --sport 4567 -j DROP
```

> **Segurança do teste:** bloqueamos **só** a porta 4567 entre os dois IPs. O acesso SSH (porta 22) e tudo mais continuam. Guarde na cabeça (ou num script) como **desfazer** isto, para não deixar o nó isolado — veja o Passo 4.

Espere o Galera declarar o outro lado inativo. Por padrão isso leva cerca de **15 segundos** (`evs.suspect_timeout` ~5s + `evs.inactive_timeout` ~15s):

```bash
sleep 20
```

## Passo 3: observar a perda de quórum

Agora olhe os **dois** nós. Saída real do lab:

```bash
# db-01
sudo mariadb -N -e "SHOW STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status','wsrep_local_state_comment');"
```

```
wsrep_local_state_comment   Initialized      <- NÃO é mais 'Synced'
wsrep_cluster_size          1                <- cada um só se vê
wsrep_cluster_status        non-Primary      <- perdeu o quórum
```

O db-02 mostra exatamente o mesmo (`size=1`, `non-Primary`). Ou seja: **os dois lados se recusam a ser o dono**. Ninguém aceita escrita — e é isso que impede o split-brain.

### O que a aplicação vê

Com o nó em `non-Primary`, a flag `wsrep_ready` fica **OFF**, e qualquer acesso a dados é recusado:

```bash
sudo mariadb -N -e "SHOW STATUS LIKE 'wsrep_ready';"     #  -> OFF
sudo mariadb -e "SELECT COUNT(*) FROM _ha_check.t;"      #  -> ERRO
sudo mariadb -e "INSERT INTO _ha_check.t(src) VALUES('x');"  #  -> ERRO
```

O erro que o cliente recebe é:

```
ERROR 1047 (08S01): WSREP has not yet prepared node for application use
```

> **Pegadinha didática:** um `SELECT 1` (constante, sem tabela) **ainda responde**, porque não toca em dados. Isso engana muita gente em plantão: "o banco responde `SELECT 1`, então está bom" — **não está**. O teste honesto é ler/escrever numa tabela, ou simplesmente olhar `wsrep_ready` e `wsrep_cluster_status`.

## Passo 4: curar a partição

Remova as regras de `iptables` (exatamente as que você criou):

```bash
# No db-02
sudo iptables -D INPUT  -s 192.168.1.21 -p tcp --dport 4567 -j DROP
sudo iptables -D OUTPUT -d 192.168.1.21 -p tcp --dport 4567 -j DROP
sudo iptables -D INPUT  -s 192.168.1.21 -p tcp --sport 4567 -j DROP

# confirme que não sobrou nenhuma regra envolvendo o db-01
sudo iptables -S | grep '192.168.1.21' || echo 'sem regras residuais'
```

## Passo 5: a recuperação (neste caso, automática)

Como **nenhum** lado aceitou escrita durante a partição, os dois continuaram com dados idênticos. Quando a rede volta, eles se reencontram e **reconstituem o Primary sozinhos**:

```bash
# db-01, ~10s depois de curar
sudo mariadb -N -e "SHOW STATUS WHERE Variable_name IN ('wsrep_cluster_size','wsrep_cluster_status','wsrep_local_state_comment');"
```

```
wsrep_local_state_comment   Synced
wsrep_cluster_size          2
wsrep_cluster_status        Primary
```

Religue o árbitro para voltar aos 3 membros:

```bash
# No db-03
sudo systemctl restart garb
#  -> wsrep_cluster_size = 3 / Primary
```

> **Por que foi automático aqui?** Porque a partição foi "limpa": os dois lados eram a membresia completa do último Primary e tinham os mesmos dados. Ao se reconectarem, o Galera reconhece o grupo e restaura o Primary. **Nem sempre** é assim — veja a seção seguinte.

## Quando a recuperação NÃO é automática (bootstrap manual do Primary)

Se, por algum motivo, o cluster ficar `non-Primary` e **não** voltar sozinho (por exemplo, um único nó sobrevivente que você tem certeza que tem os dados mais recentes), você força esse nó a reassumir o Primary:

```bash
# SOMENTE no nó que você confirmou ter os dados mais recentes
sudo mariadb -e "SET GLOBAL wsrep_provider_options='pc.bootstrap=YES';"
```

Depois disso ele vira `Primary` e os demais reentram ao iniciar o MariaDB normalmente.

> **Perigo:** `pc.bootstrap=YES` no nó errado ressuscita dados velhos e pode perder transações. Só use quando tiver certeza de qual nó é o bom (o mesmo raciocínio do `--wsrep-recover` em [galera-recuperacao-queda-total.md](galera-recuperacao-queda-total.md)). Se houver dúvida, **não** force — peça ajuda.

## Por que o árbitro resolve isso

Repita o experimento mental **com o árbitro ligado**:

- 3 membros (db-01, db-02, árbitro). Maioria = 2.
- Cai o db-02 → sobram db-01 + árbitro = **2 de 3 = maioria** → o db-01 **continua `Primary`** e aceita escrita.

Ou seja: o árbitro transforma um número par (2) de nós de dados num total ímpar (3) de votos, eliminando o empate que trava o cluster. Ele não guarda dados — só vota. Por isso roda numa VM mínima.

| Configuração | Perde 1 nó de dados | Resultado |
|---|---|---|
| 2 nós, **sem** árbitro | 1 de 2 | **non-Primary** (trava) |
| 2 nós **+ árbitro** | 2 de 3 | **Primary** (continua) |

## Resumo

- `non-Primary` **não é um erro** — é o Galera se recusando a corromper seus dados.
- O sinal honesto de saúde é `wsrep_cluster_status = Primary` **e** `wsrep_ready = ON`. `SELECT 1` não serve de teste.
- Partição limpa (sem escrita nos dois lados) → recuperação automática ao curar.
- O árbitro existe justamente para impedir o empate de 2 nós. Mantê-lo no ar é o que garante que **a perda de um nó de dados não derrube o serviço**.

## O que NÃO fazer

- **Não force `pc.bootstrap=YES` nos dois lados de uma partição.** Isso cria dois Primary independentes = split-brain real = dados divergentes.
- **Não valide saúde com `SELECT 1`.** Olhe `wsrep_cluster_status` e `wsrep_ready`.
- **Não deixe regras de `iptables` para trás.** Sempre confirme com `sudo iptables -S | grep <IP>` depois do teste.
- **Não rode o cluster permanentemente sem o árbitro.** Dois nós sem árbitro = qualquer falha de um vira indisponibilidade total.

## Veja também

- [galera-ha-haproxy-keepalived.md](galera-ha-haproxy-keepalived.md) — instalação e topologia.
- [galera-testes-de-falha-failover.md](galera-testes-de-falha-failover.md) — falhas de um componente com o cluster seguindo de pé.
- [galera-recuperacao-queda-total.md](galera-recuperacao-queda-total.md) — quando o cluster inteiro cai.
