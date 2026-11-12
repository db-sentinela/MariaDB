# MariaDB: problemas de conexão (Too many connections)

O temido `ERROR 1040: Too many connections` e como diagnosticar: quantas conexões há, quem está usando, por que a aplicação não consegue mais conectar e como sair do aperto sem derrubar o banco.

> Procedimento **executado e validado** em MariaDB 12.3.3 (nó do cluster Galera, db-01). Saídas reais. Veja também: [onde procurar](troubleshooting-diagnostico-onde-procurar.md).

## Para quem é este documento

Estagiários e juniores. A meta: diante de "a aplicação não conecta no banco", você saber se é saturação de conexões, quem está segurando e como agir.

## Conceito

Cada conexão de cliente ocupa uma "vaga". O limite é `max_connections`. Quando todas estão ocupadas, novas conexões recebem:

```
ERROR 1040 (08004): Too many connections
```

Há **uma vaga extra reservada** para um usuário com privilégio `SUPER`/`CONNECTION ADMIN` — é o que te permite entrar para investigar **mesmo com o banco "cheio"**. Guarde isso: o `root` pelo socket quase sempre consegue entrar.

## 1. Ver a situação das conexões

```sql
SELECT @@max_connections;                  -- o limite
SHOW STATUS LIKE 'Threads_connected';      -- conectadas agora
SHOW STATUS LIKE 'Max_used_connections';   -- pico desde o boot
```

No lab, o estado normal era `max_connections = 151`, com pouquíssimas conexões em uso.

## 2. Reproduzindo a saturação (o que acontece de verdade)

Para demonstrar, reduzimos o limite e abrimos conexões demais:

```sql
SET GLOBAL max_connections = 20;           -- limite baixo, de propósito
-- e abrimos 25 conexões 'carga' que ficam em SELECT SLEEP(60)
```

Observando **como root (que entra pela vaga reservada)**:

```sql
SHOW STATUS LIKE 'Threads_connected';              -- 21
SHOW STATUS LIKE 'Max_used_connections';           -- 21
SHOW STATUS LIKE 'Connection_errors_max_connections'; -- 7
SHOW STATUS LIKE 'Aborted_connects';               -- 9
```

Repare: **`Threads_connected = 21` com `max_connections = 20`** — as 20 vagas normais estão com o usuário `carga`, e a 21ª é o `root` usando a **vaga reservada de SUPER**. As conexões que sobraram (`Connection_errors_max_connections = 7`) foram recusadas.

Uma nova conexão do usuário comum falha, como esperado:

```
$ mariadb -h 192.168.1.21 -u carga -p... -e 'SELECT 1'
ERROR 1040 (08004): Too many connections
```

## 3. Diagnóstico: quem está usando as conexões?

```sql
-- Quantas por usuário
SELECT user, COUNT(*) AS conexoes
FROM information_schema.PROCESSLIST
GROUP BY user ORDER BY conexoes DESC;

-- Conexões ociosas há muito tempo (candidatas a vazamento)
SELECT id, user, host, db, time, state
FROM information_schema.PROCESSLIST
WHERE command = 'Sleep' AND time > 60
ORDER BY time DESC;
```

Muitas conexões em `Sleep` com `time` alto = a aplicação **abre e não fecha** (pool mal configurado, conexões vazando). Esse costuma ser o verdadeiro problema — não o `max_connections` baixo.

Contadores úteis:

```sql
SHOW STATUS LIKE 'Aborted_connects';   -- falhas de handshake (auth, rede)
SHOW STATUS LIKE 'Aborted_clients';    -- clientes que fecharam sem QUIT (timeout, crash)
SHOW STATUS LIKE 'Connection_errors_%';
```

## 4. Sair do aperto

1. **Entre como root pelo socket** (vaga reservada) para poder agir:
   ```bash
   sudo mariadb
   ```
2. **Identifique e mate** conexões ociosas/zumbis:
   ```sql
   KILL <id>;   -- para cada conexão Sleep antiga indevida
   ```
3. **Alívio temporário** (até achar a causa): suba o limite em runtime:
   ```sql
   SET GLOBAL max_connections = 300;
   ```
   > É paliativo. Se as conexões vazam, mais vagas só adiam o problema (e consomem memória).
4. **Corrija a causa:** pool de conexões da aplicação (tamanho e *timeout*), e o `wait_timeout` do servidor para derrubar conexões ociosas:
   ```sql
   SHOW VARIABLES LIKE 'wait_timeout';     -- padrão 28800s (8h) é alto
   ```

## 5. Dimensionar `max_connections`

- Cada conexão consome memória (buffers por sessão). Aumentar sem critério pode levar a OOM.
- Regra prática: dimensione pelo **pool da aplicação** (nº de apps × tamanho do pool) + folga + monitoramento, não por "chute alto".
- Acompanhe `Max_used_connections` ao longo do tempo para ver o pico real.

## Nota Galera

`max_connections` é **por nó** (não é do cluster). Pela topologia com HAProxy, a aplicação conecta no VIP e cai em um nó; dimensione cada nó para o tráfego que ele recebe. As conexões internas do cluster (`system user`, aplicadores wsrep) **não** contam contra o `max_connections`. Lembre que o **health check (porta 9200)** precisa conseguir conectar — se o banco satura, o health check pode falhar e o HAProxy tira o nó de rotação (ver [monitoramento do Galera](../Galera/galera-monitoramento-observabilidade.md)).

## Roteiro rápido

```sql
-- 1. Quão cheio está?
SELECT @@max_connections; SHOW STATUS LIKE 'Threads_connected';
-- 2. Quem está usando?
SELECT user, COUNT(*) FROM information_schema.PROCESSLIST GROUP BY user ORDER BY 2 DESC;
-- 3. Ociosas antigas?
SELECT id,user,time,command FROM information_schema.PROCESSLIST WHERE command='Sleep' AND time>60;
-- 4. Agir: KILL <id> das zumbis; se precisar, SET GLOBAL max_connections=<maior>.
```

## O que NÃO fazer

- **Não saia subindo `max_connections` como solução.** Quase sempre a causa é conexão vazando na aplicação.
- **Não mate conexões no escuro.** Confirme no `PROCESSLIST` que são ociosas/zumbis, não transações ativas da aplicação.
- **Não esqueça o `root` pelo socket** — é o seu "paraquedas" para entrar com o banco cheio.

## Veja também

- [Onde procurar / diagnóstico geral](troubleshooting-diagnostico-onde-procurar.md)
- [Monitoramento do Galera](../Galera/galera-monitoramento-observabilidade.md)
