# DB Sentinel | MariaDB — Galera

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração de bancos de dados.

Esta subpasta cobre **alta disponibilidade com MariaDB Galera** — instalação, failover, recuperação, monitoramento e operação de um cluster Galera de 2 nós com árbitro, HAProxy e Keepalived. Os procedimentos são independentes de clientes, não contêm dados de ambientes reais e são validados em laboratório antes da publicação.

## Procedimentos

| Procedimento | Descrição | Status |
|---|---|---|
| [Galera HA com HAProxy e Keepalived](galera-ha-haproxy-keepalived.md) | Instalação e topologia: cluster de 2 nós + árbitro, IP virtual único (3306 escrita / 3307 leitura) | Validado |
| [Testes de falha e failover](galera-testes-de-falha-failover.md) | Queda de nó de dados, failover de escrita, migração do VIP (Keepalived) e queda do árbitro | Validado |
| [Recuperação de queda total](galera-recuperacao-queda-total.md) | Cluster todo parado: diagnóstico, escolha do nó e bootstrap seguro (`safe_to_bootstrap`) | Validado |
| [Perda de quórum e recuperação](galera-perda-de-quorum-recuperacao.md) | Partição de rede, proteção contra split-brain e volta ao `Primary` | Validado |
| [Monitoramento e observabilidade](galera-monitoramento-observabilidade.md) | Variáveis `wsrep`, flow control, estatísticas do HAProxy, health check e o que alertar | Validado |
| [Operação do dia a dia](galera-operacao-dia-a-dia.md) | Rolling restart sem downtime, backup com `mariadb-backup`, adicionar/remover nó | Validado |

## Ambiente de homologação

| Item | Versão |
|---|---|
| Sistema operacional | Ubuntu 26.04.1 LTS |
| MariaDB Server | 12.3.3 |
| Galera (provider e árbitro) | 26.4.27 |
| HAProxy / Keepalived | pacotes do Ubuntu 26.04 |
| Virtualização | Oracle VirtualBox |

Cada procedimento informa as versões exatas que usa.

## Ordem de leitura sugerida

1. **Instalação** — monte o cluster ([galera-ha-haproxy-keepalived.md](galera-ha-haproxy-keepalived.md)).
2. **Monitoramento** — aprenda a ler a saúde do cluster ([galera-monitoramento-observabilidade.md](galera-monitoramento-observabilidade.md)).
3. **Testes de falha** — veja a HA reagindo a cada falha ([galera-testes-de-falha-failover.md](galera-testes-de-falha-failover.md)).
4. **Perda de quórum** — entenda o split-brain e o papel do árbitro ([galera-perda-de-quorum-recuperacao.md](galera-perda-de-quorum-recuperacao.md)).
5. **Queda total** — o pior caso e como voltar ([galera-recuperacao-queda-total.md](galera-recuperacao-queda-total.md)).
6. **Operação** — a rotina do dia a dia ([galera-operacao-dia-a-dia.md](galera-operacao-dia-a-dia.md)).

## Padrão dos documentos

Todo procedimento desta subpasta segue o mesmo formato:

- Topologia desenhada em diagrama (no procedimento de instalação).
- Versões e pré-requisitos explícitos.
- Passos numerados, com comandos prontos para copiar.
- Resultado esperado após cada etapa, com a **saída real** observada no laboratório.
- Nos procedimentos de falha e operação: conceito → comando → saída real → interpretação → **como reverter** → **o que NÃO fazer**.
- Somente o que foi validado.

## Estrutura

```
galera/
├── README.md
├── galera-ha-haproxy-keepalived.md       # instalação e topologia
├── galera-monitoramento-observabilidade.md
├── galera-testes-de-falha-failover.md
├── galera-perda-de-quorum-recuperacao.md
├── galera-recuperacao-queda-total.md
└── galera-operacao-dia-a-dia.md
```

## Como usar

1. Leia a seção de versões e pré-requisitos do procedimento.
2. Execute os passos na ordem indicada.
3. Valide cada etapa antes de seguir para a próxima.
4. Confirme a saúde do cluster pelo HAProxy (`stats;csv`), não só pelo health check local.
5. Troque todas as senhas de exemplo antes de usar em qualquer ambiente real.

## Aviso

Os procedimentos foram validados em laboratório. Vários incluem ações destrutivas controladas (parar nós, derrubar serviços, simular partição de rede) — execute-os em ambiente de homologação antes de aplicar em produção, e sempre com um nó por vez.

## Mantenedor

DB Sentinel Consultoria em Tecnologia da Informacao Ltda.
