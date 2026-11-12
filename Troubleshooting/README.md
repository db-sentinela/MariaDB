# DB Sentinel | MariaDB — Troubleshooting

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração de bancos de dados.

Esta subpasta cobre **troubleshooting de MariaDB** — identificar e resolver bloqueios por lock, entender e ler deadlocks, e saber onde procurar cada informação de diagnóstico. Os procedimentos são independentes de clientes, não contêm dados de ambientes reais e são validados em laboratório antes da publicação.

## Procedimentos

| Procedimento | Descrição | Status |
|---|---|---|
| [Locks e bloqueios](troubleshooting-locks-e-bloqueios.md) | Como ver quem bloqueia quem (row lock e metadata lock), e resolver com `KILL`; inclui o comportamento específico de DDL em Galera (BF-abort) | Validado |
| [Deadlocks](troubleshooting-deadlocks.md) | Como um deadlock nasce, ler o relatório `LATEST DETECTED DEADLOCK`, registrar no log e prevenir | Validado |
| [Diagnóstico — onde procurar](troubleshooting-diagnostico-onde-procurar.md) | Mapa de "pergunta → onde olhar": processlist, `INNODB_TRX`, lock waits, metadata locks, logs, `sys` schema | Validado |
| [Queries lentas](troubleshooting-queries-lentas.md) | Slow log, `EXPLAIN`/`ANALYZE` e correção por índice (full scan → ref); teste em tabela de 500k linhas | Validado |
| [Problemas de conexão](troubleshooting-conexoes.md) | `Too many connections` (1040): diagnosticar saturação, vaga reservada de SUPER e conexões vazando | Validado |
| [Galera: nó não entra (SST/IST)](troubleshooting-galera-sst-join.md) | State transfer: IST x SST, SST completo forçado e diagnóstico de falhas de join | Validado |

## Ambiente de homologação

| Item | Versão |
|---|---|
| Sistema operacional | Ubuntu 26.04.1 LTS |
| MariaDB Server | 12.3.3 |
| Topologia | nó do cluster Galera (ver subpasta `../Galera`) |
| Virtualização | Oracle VirtualBox |

Os cenários foram executados num nó do cluster Galera. Onde o comportamento difere de um servidor standalone (ex.: DDL que aborta transações em vez de esperar o metadata lock), o documento destaca a diferença.

## Padrão dos documentos

- Conceito → montagem do cenário → comando → **saída real** observada no laboratório → interpretação.
- Como **resolver** e o que **NÃO fazer**.
- Somente o que foi validado.

## Estrutura

```
troubleshooting/
├── README.md
├── troubleshooting-locks-e-bloqueios.md
├── troubleshooting-deadlocks.md
├── troubleshooting-diagnostico-onde-procurar.md
├── troubleshooting-queries-lentas.md
├── troubleshooting-conexoes.md
└── troubleshooting-galera-sst-join.md
```

## Objetos de teste

Os cenários usam um schema descartável `lab_tshoot` (tabela `contas` para locks/deadlocks; tabela `eventos` com 500 mil linhas para performance), criado e removido durante os testes. Nenhum objeto de teste fica no servidor após a publicação.

## Como usar

1. Comece por [Diagnóstico — onde procurar](troubleshooting-diagnostico-onde-procurar.md) para ter o mapa geral.
2. Use [Locks e bloqueios](troubleshooting-locks-e-bloqueios.md) quando uma query estiver "pendurada".
3. Use [Deadlocks](troubleshooting-deadlocks.md) quando aparecer o erro `1213`.
4. Reproduza os cenários em homologação antes de confiar no procedimento em produção.

## Aviso

Os procedimentos foram validados em laboratório. Vários criam concorrência artificial (transações abertas, `KILL`) — execute em ambiente de homologação antes de aplicar em produção.

## Mantenedor

DB Sentinel Consultoria em Tecnologia da Informacao Ltda.
