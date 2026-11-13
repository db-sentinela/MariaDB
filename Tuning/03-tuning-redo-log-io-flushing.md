# 03 — Redo Log, I/O Capacity e Frequência de Flush (ACID)

O mecanismo de **Write-Ahead Logging (WAL)** do InnoDB assegura a durabilidade das transações através do Redo Log (`ib_logfile`). A política de sincronização em disco dessas gravações (`innodb_flush_log_at_trx_commit`) e a calibração de I/O (`innodb_io_capacity`) determinam diretamente a capacidade de vazão (*throughput*) de escrita e a latência de confirmação (*commit*) do banco. Em clusters Galera, essa equação possui particularidades críticas que impactam a performance sem comprometer a resiliência.

---

## O que é

Quando uma transação realiza modificações (`INSERT`, `UPDATE`, `DELETE`), os dados são alterados nas páginas do Buffer Pool e um registro sequencial da operação é escrito no **Redo Log Buffer** em memória. Para que a transação seja considerada durável perante uma falha de energia, esses registros precisam ser persistidos nos arquivos de Redo Log no disco.

O comportamento do commit é controlado pela variável `innodb_flush_log_at_trx_commit`:
- **Modo 1 (Full ACID - Padrão):** A cada `COMMIT`, o log buffer é gravado no arquivo de log do SO e uma operação de `fsync()` é chamada para descarregar o cache do controlador físico para o disco. Garante tolerância total a panes no SO ou falhas de energia.
- **Modo 2 (Flush no OS Cache, fsync a cada 1s):** A cada `COMMIT`, o buffer é transferido para o buffer cache do kernel do sistema operacional, mas a chamada bloqueante de `fsync()` em disco físico só ocorre uma vez por segundo.
- **Modo 0 (Flush e fsync a cada 1s):** O log buffer permanece no processo `mariadbd` e a gravação no SO mais o `fsync` ocorrem uma vez por segundo através de uma thread interna.

Em conjunto, `innodb_io_capacity` e `innodb_io_capacity_max` definem o limite de IOPS que as threads de background do InnoDB podem utilizar para descarregar páginas sujas (*flush*) e mesclar inserções.

---

## Por que importa

1. **Eliminação do Gargalo de Latência de Fsync:** A chamada de sistema `fsync()` é síncrona e bloqueante. Em transações curtas em lote (OLTP de alta frequência), esperar o disco responder a cada commit limita a aplicação a algumas centenas de transações por segundo.
2. **A "Trapaça Segura" do Galera Cluster:** Em uma arquitetura de banco de dados standalone, usar `flush=2` expõe o servidor a perder até 1 segundo de transações se houver corte abrupto de energia na máquina. Porém, no **Galera Cluster**, a transação já foi sincronizada e certificada na memória e nó dos outros membros do cluster antes de responder ao cliente. Logo, usar `flush=2` no Galera é prática amplamente recomendada em produção para elevar a vazão sem risco de perda de dados.
3. **Prevenção de I/O Stalls:** Se `innodb_io_capacity` for mantido no valor padrão arcaico (200 IOPS) em um subsistema moderno de SSD NVMe que suporta 50.000 IOPS, o InnoDB demorará para descarregar páginas sujas, atingirá o teto de dirty pages e pausará todas as consultas para realizar *furious flushing*.

---

## Como reproduzir

Realize o benchmark comparativo e a calibração dos parâmetros de I/O no nó `mariadb-db-01`:

### 1. Criar o schema e procedure de benchmark
```sql
CREATE DATABASE IF NOT EXISTS lab_tuning;
USE lab_tuning;

CREATE OR REPLACE TABLE t_bench (
    id INT AUTO_INCREMENT PRIMARY KEY,
    dt TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    val VARCHAR(50)
) ENGINE=InnoDB;

DELIMITER //
CREATE OR REPLACE PROCEDURE p_bench(IN total INT)
BEGIN
    DECLARE i INT DEFAULT 0;
    WHILE i < total DO
        INSERT INTO t_bench (val) VALUES (CONCAT('val_', i));
        SET i = i + 1;
    END WHILE;
END //
DELIMITER ;
```

### 2. Testar o Modo 1 (Full ACID com fsync síncrono a cada commit)
```sql
SET GLOBAL innodb_flush_log_at_trx_commit = 1;
```

Execute o teste cronometrado no terminal bash:
```bash
t1_start=$(date +%s%N)
mariadb -e "USE lab_tuning; CALL p_bench(1000);"
t1_end=$(date +%s%N)
echo "Tempo (flush=1): $(( (t1_end - t1_start) / 1000000 )) ms"
```

### 3. Testar o Modo 2 (Escrita no OS Cache, fsync periódico de 1 segundo)
```sql
SET GLOBAL innodb_flush_log_at_trx_commit = 2;
```

Execute o mesmo lote de 1.000 inserções:
```bash
t2_start=$(date +%s%N)
mariadb -e "USE lab_tuning; CALL p_bench(1000);"
t2_end=$(date +%s%N)
echo "Tempo (flush=2): $(( (t2_end - t2_start) / 1000000 )) ms"
```

### 4. Inspecionar métricas de I/O, Redo Log e Contenção
```sql
SHOW VARIABLES WHERE Variable_name IN (
    'innodb_flush_log_at_trx_commit',
    'innodb_log_file_size',
    'innodb_log_buffer_size',
    'innodb_io_capacity',
    'innodb_io_capacity_max'
);

SHOW STATUS WHERE Variable_name IN (
    'Innodb_os_log_written',
    'Innodb_os_log_fsyncs',
    'Innodb_log_waits'
);
```

---

## O que você vê (saída real)

### Benchmark de Latência de Commit (1.000 transações unitárias):
```text
=== Teste 1: innodb_flush_log_at_trx_commit = 1 (Full ACID fsync) ===
Tempo decorrido (flush=1): 6456 ms

=== Teste 2: innodb_flush_log_at_trx_commit = 2 (OS cache write, fsync 1s) ===
Tempo decorrido (flush=2): 5846 ms
```
*(No cluster virtualizado, a transação com `flush=2` obteve ganho direto de tempo; em discos físicos tradicionais ou storage de rede SAN/NFS, a redução de latência com `flush=2` costuma ultrapassar 300% a 500%).*

### Variáveis e Métricas de Redo Log:
```text
+--------------------------------+-----------+
| Variable_name                  | Value     |
+--------------------------------+-----------+
| innodb_flush_log_at_trx_commit | 2         |
| innodb_io_capacity             | 200       |
| innodb_io_capacity_max         | 2000      |
| innodb_log_buffer_size         | 16777216  |
| innodb_log_file_size           | 100663296 |
+--------------------------------+-----------+

+-----------------------+--------+
| Variable_name         | Value  |
+-----------------------+--------+
| Innodb_log_waits      | 0      |
| Innodb_os_log_written | 885018 |
+-----------------------+--------+
```

---

## Lições

- **Recomendação no Galera Cluster:** Em clusters MariaDB Galera síncronos, utilize `innodb_flush_log_at_trx_commit = 2`. Como o writeset já está garantido em múltiplos nós físicos antes do retorno da transação, você não compromete a consistência e ganha enorme alívio no subsistema de disco local.
- **Dimensionamento do `innodb_log_file_size`:** No MariaDB 12, o Redo Log deve ser dimensionado para reter entre **1 e 2 horas de tráfego de escrita do pico**. Para estimar isso:
  1. Capture `Innodb_os_log_written` no início do minuto.
  2. Aguarde 60 segundos e capture novamente.
  3. Calcule: $\text{Bytes por hora} = (\text{Valor}_2 - \text{Valor}_1) \times 60$.
  4. Se o resultado for 2 GB por hora, configure `innodb_log_file_size = 2G`.
- **Ajuste de `innodb_io_capacity`:**
  - Discos magnéticos legados (HDD 7200 RPM): 200
  - SSDs SATA corporativos: 1.000 a 3.000 (com `max` em 6.000)
  - SSDs NVMe modernos: 10.000 a 25.000 (com `max` em 50.000)

---

## Pegadinhas

- **O Mito do `flush=0`:** O valor `0` faz o processo do MariaDB segurar o log em sua própria memória interna. Se o processo morrer por um `kill -9` ou *segmentation fault*, dados comitados são perdidos, mesmo que o SO não tenha reiniciado. O valor `2` é infinitamente mais seguro pois os dados já estão no buffer do kernel do Linux.
- **Indicador `Innodb_log_waits > 0`:** Se este status for maior que zero, significa que o Redo Log Buffer (`innodb_log_buffer_size`, 16 MB por padrão) esgotou o espaço antes que o flusher conseguisse esvaziar, forçando transações a esperarem. Aumente para 32 MB ou 64 MB imediatamente caso ocorra.
- **Discos com Write-Back Cache sem BBU (Battery Backup Unit):** Nunca confie cegamente em `flush=1` se o seu storage ou controladora RAID física estiver configurada com *Write-Back Cache* sem bateria ou capacitor. Uma queda de energia apagará os dados da controladora antes que cheguem aos pratos/células flash.
