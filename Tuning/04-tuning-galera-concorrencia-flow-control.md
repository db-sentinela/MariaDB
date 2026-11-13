# 04 — Galera Cluster: Concorrência de Replicação, Flow Control e Gcache

O MariaDB Galera Cluster utiliza replicação síncrona baseada em certificação (*Certification-based Replication*). Para que o cluster atinja alta performance e não sofra paradas súbitas sob carga mista, três pilares de tuning são indispensáveis: o paralelismo de aplicação de writesets (**Applier Threads**), a tolerância a atrasos de replicação (**Flow Control**) e a retenção do buffer circular de transações (**Gcache**).

---

## O que é

No Galera, quando um nó originador recebe um `COMMIT`, ele encapsula todas as modificações da transação em um pacote chamado **writeset** e o transmite via multicast (`gcomm`) para todos os nós do cluster.
1. **Certification:** Cada nó valida de forma atômica se o writeset conflita com transações locais não confirmadas.
2. **Apply (Applier Threads):** Uma vez certificado, o writeset deve ser gravado nas tabelas locais. O parâmetro `wsrep_slave_threads` define quantas threads paralelas aplicam esses writesets simultaneamente.
3. **Flow Control:** Se um nó secundário ficar sobrecarregado (seja por contenção de disco, CPU insuficiente ou poucas applier threads), sua fila de entrada (`recv_queue`) começa a crescer. Para evitar que o nó fique desincronizado indefinidamente, o Galera ativa o **Flow Control**, pausando temporariamente as escritas em **todos os nós do cluster**. Os parâmetros `gcs.fc_limit` e `gcs.fc_factor` determinam o gatilho dessa pausa.
4. **Gcache (Galera Cache):** É um buffer circular em disco e memória (`galera.cache`, padrão 512 MB). Ele armazena os writesets gerados pelo cluster. Quando um nó cai ou é reiniciado para manutenção, ao retornar ele solicita um **IST (Incremental State Transfer)**. Se a transação mais antiga que ele precisa ainda estiver no Gcache dos outros nós, ele se sincroniza em segundos sem interromper ninguém. Se o Gcache tiver sido reciclado, ele é forçado a um **SST (Snapshot State Transfer)** completo e custoso.

---

## Por que importa

1. **Prevenção de Incidentes por Flow Control Paused:** Se o `gcs.fc_limit` for muito baixo (o padrão de fábrica é 16) e houver uma carga massiva de escritas, rajadas momentâneas na rede acionam o Flow Control. O status `wsrep_flow_control_paused` sobe e **toda a aplicação congela**, gerando erros de timeout HTTP e conexões represadas.
2. **Eficácia de Janelas de Manutenção Sem SST:** Um Gcache mal dimensionado obriga qualquer nó que fique 10 minutos fora do ar a sofrer um SST completo (transferência de todos os gigabytes do banco via `mariadb-backup` ou `rsync`), sobrecarregando a rede e o nó doador. Com Gcache bem calibrado, a recuperação ocorre 100% via IST em poucos segundos.
3. **Aproveitamento de CPU em Nós Secundários:** O MariaDB 12 possui algoritmos aprimorados de dependência de transações (`cert.optimistic_pa`). Configurar `wsrep_slave_threads` com base no hardware permite que nós réplicas apliquem writesets com a mesma agilidade que o nó primário grava.

---

## Como reproduzir

Execute os passos a seguir no laboratório para calibrar threads, inspecionar filas e entender as limitações dinâmicas do Galera 4:

### 1. Inspecionar o paralelismo atual e o histórico de Flow Control
```sql
SELECT @@wsrep_slave_threads;

SHOW STATUS WHERE Variable_name IN (
    'wsrep_applier_threads',
    'wsrep_flow_control_paused',
    'wsrep_local_recv_queue_avg',
    'wsrep_local_recv_queue_max'
);
```

### 2. Ajustar dinamicamente o paralelismo de replicação
O número de threads de aplicação de writesets pode ser aumentado em tempo de execução sem reiniciar o serviço:
```sql
-- Eleva de 4 para 8 threads no db-01
SET GLOBAL wsrep_slave_threads = 8;

-- Validar a aplicacao imediata
SELECT @@wsrep_slave_threads;
```

### 3. A Pegadinha do Flow Control Dinâmico no Galera 4
Tente alterar as opções do provider em runtime:
```sql
SET GLOBAL wsrep_provider_options = 'gcs.fc_limit=64;gcs.fc_factor=0.8';
```
*(Observe que no MariaDB 12 com Galera 4, essa variável é protegida e não aceita modificação com o provider carregado, disparando o erro `ERROR 1210`).*

### 4. Configurar permanentemente no arquivo de configuração
Para aplicar o Flow Control e o Gcache calibrados, edite `/etc/mysql/mariadb.conf.d/60-galera.cnf` em cada nó:

```ini
[mariadb]
# Paralelismo de aplicacao (2x a 4x o total de vCPUs)
wsrep_slave_threads = 8

# Opcoes avancadas de replicacao e horizonte de cache
wsrep_provider_options = "gcache.size=1G;gcs.fc_limit=64;gcs.fc_factor=0.8"
```

### 5. Verificar a capacidade e horizonte do Gcache
Consulte os limites de seqno mantidos atualmente no cache:
```sql
SHOW STATUS WHERE Variable_name IN (
    'wsrep_last_committed',
    'wsrep_local_cached_downto'
);
```
*(A diferença `wsrep_last_committed - wsrep_local_cached_downto` indica a quantidade exata de transações disponíveis no cache para atender nós atrasados).*

---

## O que você vê (saída real)

### Status inicial de concorrência e filas no cluster:
```text
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| @@wsrep_slave_threads | 4     |
+-----------------------+-------+

+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| wsrep_flow_control_paused  | 0     |
| wsrep_local_recv_queue_avg | 0.1   |
| wsrep_local_recv_queue_max | 2     |
+----------------------------+-------+
```

### Ajuste dinâmico de `wsrep_slave_threads` executado com sucesso:
```sql
mariadb -e "SET GLOBAL wsrep_slave_threads = 8;"
mariadb -e "SELECT @@wsrep_slave_threads;"
```
Saída real:
```text
+-----------------------+
| @@wsrep_slave_threads |
+-----------------------+
| 8                     |
+-----------------------+
```

### Tentativa de alterar `wsrep_provider_options` em runtime no Galera 4:
```text
mariadb -e "SET GLOBAL wsrep_provider_options = 'gcs.fc_limit=64;gcs.fc_factor=0.8';"
ERROR 1210 (HY000) at line 1: wsrep_provider_options cannot be changed while the wsrep-provider plugin is loaded
```

### Horizonte de retenção no Gcache:
```sql
SHOW STATUS WHERE Variable_name IN ('wsrep_last_committed', 'wsrep_local_cached_downto');
```
Saída real:
```text
+--------------------------+-------+
| Variable_name            | Value |
+--------------------------+-------+
| wsrep_last_committed     | 2217  |
| wsrep_local_cached_downto| 55    |
+--------------------------+-------+
```
*(O nó retém todas as transações desde o seqno 55 até 2217 no buffer circular).*

---

## Lições

- **Fórmula de Dimensionamento de `wsrep_slave_threads`:**
  
  $$\text{wsrep\_slave\_threads} = (2 \text{ a } 4) \times \text{vCPUs da máquina}$$
  
  Em nosso laboratório com 4 vCPUs, o valor ideal é **8 threads**. Se configurado para menos que o número de vCPUs, os nós secundários não conseguirão aplicar writesets concorrentes gerados pelo nó mestre.
- **Calibração de `gcs.fc_limit`:** O padrão (16) é extremamente sensível a oscilações normais de rede. Em servidores modernos com discos rápidos e múltiplos núcleos, aumentar para **64 ou 128** evita falsos positivos de Flow Control sem colocar em risco a memória do servidor.
- **Sizing de Gcache:** O arquivo de cache deve ser dimensionado considerando a taxa horária de geração de writesets multiplicada pelas horas que um nó pode passar em manutenção:
  
  $$\text{gcache.size} = \text{Writesets por hora (MB)} \times \text{Janela de Manutenção Desejada (horas)}$$

---

## Pegadinhas

- **`wsrep_provider_options` Estático no Galera 4 / MariaDB 12:** Enquanto no Galera 3 / MariaDB 10.3 algumas opções podiam ser alteradas em runtime, no Galera 4 a tentativa de alterar `gcs.fc_limit` ou `gcache.size` via `SET GLOBAL` resulta no erro fatal `ERROR 1210 (HY000)`. Qualquer mudança nessas opções exige planejamento, edição no arquivo de configuração e *rolling restart* do cluster.
- **A Métrica Acumuladora `wsrep_flow_control_paused`:** Este contador **não** é uma taxa por segundo; ele é um número de ponto flutuante cumulativo que mede a fração de tempo em que o nó esteve pausado desde o início do serviço. Em ferramentas de monitoramento (como o PMM), você deve sempre calcular a **taxa de variação (delta / per-second rate)**, e nunca alertar sobre o valor absoluto. Se a taxa de variação for maior que 0.05 (5% do tempo em pausa), há estrangulamento severo.
- **Espaço em Disco do Gcache Pré-Alocado:** O arquivo `galera.cache` em `/var/lib/mysql/` é pré-alocado em disco no momento da subida do serviço. Se você configurar `gcache.size = 20G` em uma partição que possui apenas 15 GB livres, o MariaDB falhará ao iniciar.
