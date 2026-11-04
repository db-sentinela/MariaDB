# DB Sentinel | MariaDB

Parte do **DB Sentinel Database Procedures Framework**: biblioteca técnica de procedimentos operacionais (SOPs) para administração de bancos de dados.

Este repositório cobre **MariaDB**. Os procedimentos são independentes de clientes, não contêm dados de ambientes reais e são validados em laboratório antes da publicação.

## Procedimentos

| Procedimento | Descrição | Status |
|---|---|---|
| [Galera HA com HAProxy e Keepalived](galera/galera-ha-haproxy-keepalived.md) | Cluster Galera com 2 nós e árbitro, IP virtual único para a aplicação | Validado |
| Testes de falha do Galera | Queda de nó, HAProxy e árbitro | Planejado |
| Recuperação do cluster Galera | Perda de quórum e queda total | Planejado |

## Ambiente de homologação

| Item | Versão |
|---|---|
| Sistema operacional | Ubuntu 26.04.1 LTS |
| MariaDB Server | 12.3.3 |
| Galera | 26.4.27 |
| Virtualização | Oracle VirtualBox |

Cada procedimento informa as versões exatas que usa.

## Padrão dos documentos

Todo procedimento deste repositório segue o mesmo formato:

- Topologia desenhada em diagrama.
- Versões e pré-requisitos explícitos.
- Passos numerados, com comandos prontos para copiar.
- Resultado esperado após cada etapa.
- Somente o que foi validado.

## Estrutura

```
mariadb/
├── README.md
└── galera/
    └── galera-ha-haproxy-keepalived.md
```

## Como usar

1. Leia a seção de versões e pré-requisitos do procedimento.
2. Execute os passos na ordem indicada.
3. Valide cada etapa antes de seguir para a próxima.
4. Troque todas as senhas de exemplo antes de usar em qualquer ambiente real.

## Aviso

Os procedimentos foram validados em laboratório. Teste em ambiente de homologação antes de aplicar em produção.

## Mantenedor

DB Sentinel Consultoria em Tecnologia da Informacao Ltda.
