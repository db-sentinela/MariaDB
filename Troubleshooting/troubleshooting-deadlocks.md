# MariaDB: deadlocks — como identificar, ler o relatório e prevenir

Um **deadlock** é um "abraço mortal": duas transações ficam esperando uma pela outra, em círculo, e nenhuma consegue avançar. O InnoDB detecta isso automaticamente e desfaz o impasse **abortando uma das transações**. Este documento mostra como reconhecer, ler o relatório e evitar.

> Procedimento **executado e validado** em MariaDB 12.3.3 (cluster Galera, db-01). Saídas reais. Veja também: [locks e bloqueios](troubleshooting-locks-e-bloqueios.md) e [onde procurar](troubleshooting-diagnostico-onde-procurar.md).

## Para quem é este documento

Estagiários e juniores. A meta: entender que o erro `1213` **não é um bug do banco** — é o InnoDB te protegendo — e saber ler o relatório para descobrir **por que** aconteceu.

## Deadlock x bloqueio simples

- **Bloqueio simples:** A segura, B espera. B destrava quando A commita (ou estoura o timeout). Só uma espera. Ver [locks e bloqueios](troubleshooting-locks-e-bloqueios.md).
- **Deadlock:** A espera B **e** B espera A. Ninguém destravaria nunca — então o InnoDB **escolhe uma vítima** e a aborta na hora, com o erro:
  ```
  ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
  ```

O `40001` (SQLSTATE) é o sinal universal de "tente de novo". A transação vítima é totalmente revertida; a outra segue e commita.

## Como um deadlock nasce (o padrão clássico: ordem inversa)

Duas transações que travam as **mesmas linhas em ordem oposta**:

| Tempo | Sessão A | Sessão B |
|---|---|---|
| t1 | `UPDATE contas ... WHERE id=1` (trava a linha 1) | `UPDATE contas ... WHERE id=2` (trava a linha 2) |
| t2 | `UPDATE contas ... WHERE id=2` (espera B) | `UPDATE contas ... WHERE id=1` (espera A) |
| t3 | → **abraço mortal** → InnoDB aborta uma delas | |

SQL usado no laboratório:

```sql
-- Sessão A
START TRANSACTION;
UPDATE lab_tshoot.contas SET saldo = saldo - 10 WHERE id = 1;
-- (pausa)
UPDATE lab_tshoot.contas SET saldo = saldo + 10 WHERE id = 2;   -- espera B
COMMIT;

-- Sessão B
START TRANSACTION;
UPDATE lab_tshoot.contas SET saldo = saldo - 10 WHERE id = 2;
-- (pausa)
UPDATE lab_tshoot.contas SET saldo = saldo + 10 WHERE id = 1;   -- espera A -> DEADLOCK
```

Resultado real: a **A commitou** e a **B foi a vítima**:

```
A: COMMIT OK
B: ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```

> **Quem é escolhido como vítima?** O InnoDB aborta a transação "mais barata" de desfazer (a que modificou menos linhas / fez menos trabalho). Você não controla diretamente qual será — por isso a aplicação **sempre** deve estar preparada para repetir uma transação que recebeu `1213`.

## Lendo o relatório: `SHOW ENGINE INNODB STATUS`

O InnoDB guarda o **último** deadlock detectado. Para ver:

```sql
SHOW ENGINE INNODB STATUS\G
```

Procure a seção `LATEST DETECTED DEADLOCK`. Trecho real do lab:

```
LATEST DETECTED DEADLOCK
------------------------
2026-11-11 21:56:40
*** (1) TRANSACTION:
TRANSACTION 328, ACTIVE 3 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1120, 2 row lock(s), undo log entries 1
MariaDB thread id 75, ... query id 160 localhost root Updating
UPDATE lab_tshoot.contas SET saldo = saldo + 10 WHERE id = 1
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 16 ... index PRIMARY of table `lab_tshoot`.`contas`
   trx id 328 lock_mode X locks rec but not gap waiting
*** (2) TRANSACTION:
TRANSACTION 327, ACTIVE 3 sec starting index read
...
*** WE ROLL BACK TRANSACTION (2)
```

Como ler, linha a linha:

- **`*** (1) TRANSACTION` / `*** (2) TRANSACTION`**: as duas transações envolvidas.
- **`TRANSACTION 328, ACTIVE 3 sec`**: id interno e há quanto tempo está ativa.
- **`MariaDB thread id 75`**: a conexão (o id do `PROCESSLIST`).
- **A query mostrada**: o comando que estava **esperando** quando o impasse fechou (`UPDATE ... WHERE id = 1`).
- **`WAITING FOR THIS LOCK`**: o lock que a transação queria — aqui, `lock_mode X` (exclusivo) na `index PRIMARY` da tabela `contas`.
- **`WE ROLL BACK TRANSACTION (2)`**: qual das duas foi a vítima (a que o InnoDB reverteu).

Com isso você descobre **as duas queries** e **as linhas/índices** em conflito — é o suficiente para achar o trecho de código que precisa mudar.

## Registrar todos os deadlocks no log de erro

Por padrão o `SHOW ENGINE INNODB STATUS` guarda só o **último** deadlock. Para registrar **todos** no log de erro (ótimo para diagnóstico depois do fato):

```sql
SET GLOBAL innodb_print_all_deadlocks = ON;
```

> Vale em tempo de execução; para ficar permanente, coloque `innodb_print_all_deadlocks = ON` no `.cnf`. No MariaDB o padrão é `OFF`.

Com isso ligado, cada deadlock aparece no `journalctl` (não existe `/var/log/mysql/error.log` nesta instalação):

```bash
sudo journalctl -u mariadb --since '5 min ago' --no-pager -o cat | grep -iA2 deadlock
```

Saída real:

```
[Note] InnoDB: Transactions deadlock detected, dumping detailed information.
...
```

## Nota sobre Galera (deadlocks "que não são bem deadlocks")

Em um cluster Galera, você também pode ver o erro `1213` por um motivo diferente: **conflito de certificação / BF-abort**. Quando dois nós aplicam escritas conflitantes, o Galera aborta a transação local para preservar a consistência — e a aplicação recebe o mesmo `1213`/`40001`. Sinais:

- O contador `wsrep_local_bf_aborts` sobe (veja [monitoramento do Galera](../Galera/galera-monitoramento-observabilidade.md)).
- Também acontece quando um **DDL (TOI)** atropela uma transação local (ver o Cenário 2 em [locks e bloqueios](troubleshooting-locks-e-bloqueios.md)).

A boa notícia: o tratamento na aplicação é o **mesmo** — pegar `1213`/`40001` e repetir a transação.

## Como prevenir deadlocks

1. **Acesse as linhas sempre na mesma ordem** em todas as transações (ex.: sempre por `id` crescente). Isso sozinho elimina a maioria dos deadlocks.
2. **Transações curtas.** Quanto menos tempo segurando locks, menor a janela de colisão. Nunca deixe uma transação aberta esperando input do usuário.
3. **Índices adequados.** Sem índice, um `UPDATE ... WHERE` pode travar muito mais linhas do que o necessário, aumentando a colisão.
4. **Trate o `1213` na aplicação.** Sempre tenha lógica de *retry* (repetir a transação) ao receber `40001`. Não é opcional — é parte do contrato do InnoDB/Galera.
5. **Evite "hot rows".** Linhas atualizadas por todo mundo ao mesmo tempo (um contador global, por exemplo) são ímã de conflito.

## Referência rápida

```sql
-- Ler o último deadlock
SHOW ENGINE INNODB STATUS\G      -- seção LATEST DETECTED DEADLOCK

-- Registrar todos no log
SET GLOBAL innodb_print_all_deadlocks = ON;
```
```bash
# Ver no log de erro
sudo journalctl -u mariadb --since '10 min ago' --no-pager -o cat | grep -iA3 deadlock
```

## O que NÃO fazer

- **Não trate `1213` como erro fatal.** É esperado sob concorrência; a resposta correta é **repetir** a transação.
- **Não "resolva" deadlock aumentando timeout** — timeout é para bloqueio, não para deadlock (o InnoDB já desfaz o deadlock na hora).
- **Não ignore deadlocks recorrentes.** Se aparecem com frequência, há um padrão de acesso fora de ordem ou transações longas demais — corrija a causa.

## Veja também

- [Locks e bloqueios](troubleshooting-locks-e-bloqueios.md)
- [Onde procurar / diagnóstico geral](troubleshooting-diagnostico-onde-procurar.md)
- [Monitoramento do Galera](../Galera/galera-monitoramento-observabilidade.md)
