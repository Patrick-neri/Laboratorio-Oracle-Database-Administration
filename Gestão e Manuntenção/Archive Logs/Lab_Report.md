# LAB_REPORT — Lab: Gerenciando Archived Redo Logs

**Data de execução:** 05/07/2026  
**Ambiente:** Oracle Database 19c Enterprise Edition — Oracle Linux 7  

---

## 1. Contexto e objetivo

O modo ARCHIVELOG é o que torna possível recuperar um banco Oracle até o exato momento de uma falha (point-in-time recovery), em vez de apenas até o último backup. Sem ele, qualquer transação commitada após o backup mais recente é perdida em caso de falha de mídia.

Este lab cobre a ativação do modo ARCHIVELOG, a configuração de múltiplos destinos de arquivamento (multiplexação), e a personalização do formato de nomenclatura dos archived logs via `LOG_ARCHIVE_FORMAT`.

**Por que importa na prática:**  
Em qualquer ambiente de produção corporativo, ARCHIVELOG é praticamente obrigatório — é o que permite fazer backup com o banco aberto (sem downtime) e recuperar transações até o segundo da falha. Configurar múltiplos destinos de arquivamento é prática de resiliência: se um disco falha, o archived log ainda existe em outro local.

---

## 2. Conceitos fundamentais

### NOARCHIVELOG vs ARCHIVELOG

| | NOARCHIVELOG | ARCHIVELOG |
|---|---|---|
| Arquivamento do redo | Não ocorre | Ocorre a cada log switch |
| Protege contra | Falha de instância | Falha de instância e de mídia |
| Grupo cheio | Sobrescrito assim que INACTIVE | Fica ACTIVE até ser arquivado |
| Backup com banco aberto | Não | Sim |

### O processo ARCn

Assim que ocorre um log switch, o grupo de redo se torna elegível para arquivamento. O processo de background **ARCn** copia os registros de redo do grupo para um archived redo log — uma cópia com o mesmo log sequence number do grupo de origem.

### Os dois métodos de definir destino de arquivamento

| Método | Parâmetro | Escopo |
|---|---|---|
| 1 | `LOG_ARCHIVE_DEST_n` (n = 1 a 31) | Local ou remoto — permite múltiplos destinos numerados |
| 2 | `LOG_ARCHIVE_DEST` + `LOG_ARCHIVE_DUPLEX_DEST` | Apenas local — no máximo 2 destinos |

---

## 3. Sequência de execução

### 3.1 Verificando o estado inicial e habilitando ARCHIVELOG

```sql
SHOW PDBS;

SELECT LOG_MODE FROM V$DATABASE;
-- NOARCHIVELOG
```

> <img width="319" height="94" alt="image" src="https://github.com/user-attachments/assets/ce6e8e31-332c-45c0-acbc-dc294007a336" />

Sequência de ativação (exige o banco em MOUNT):

```sql
SHUTDOWN IMMEDIATE;
STARTUP MOUNT;
ALTER DATABASE ARCHIVELOG;
ALTER DATABASE OPEN;

SELECT LOG_MODE FROM V$DATABASE;
-- ARCHIVELOG ✅
```

> <img width="345" height="432" alt="image" src="https://github.com/user-attachments/assets/d455a3fd-8c4f-4694-9a46-13874923d825" />

Verificação com `ARCHIVE LOG LIST`:

```sql
ARCHIVE LOG LIST;
```

```
Database log mode              Archive Mode
Automatic archival             Enabled
Archive destination            USE_DB_RECOVERY_FILE_DEST
Oldest online log sequence     1
Next log sequence to archive   3
Current log sequence           3
```

> 💡 `ARCHIVE LOG LIST` é o comando mais direto para diagnóstico rápido do modo de arquivamento — mostra modo, destino padrão e sequências em uma única saída, sem precisar consultar `V$DATABASE` e `V$LOG` separadamente.

Confirmando o status dos grupos após a ativação:

```sql
SELECT GROUP#, THREAD#, BYTES/1024/1024 AS "TAMANHO MB",
       SEQUENCE#, MEMBERS, ARCHIVED, STATUS FROM V$LOG;
```

```
GROUP#  THREAD#  TAMANHO MB  SEQUENCE#  MEMBERS  ARC  STATUS
------  -------  ----------  ---------  -------  ---  ----------
     1        1         200          1        1  YES  INACTIVE
     2        1         200          2        2  YES  INACTIVE
     3        1         200          3        1  NO   CURRENT
    10        1         200          0        1  YES  UNUSED
```

> 💡 A coluna `ARCHIVED` agora reflete se aquele grupo específico já foi arquivado — grupos 1 e 2 já foram (YES), o grupo 3 (CURRENT) ainda não.

---

### 3.2 Gerando atividade e observando o arquivamento automático

```sql
CREATE TABLE TESTE (C1 NUMBER);

INSERT INTO TESTE VALUES(1);
/
/
/
COMMIT;
```

> <img width="347" height="405" alt="image" src="https://github.com/user-attachments/assets/2de461e3-49a0-409d-b54f-1260fbaefdb0" />


Forçando um log switch para observar o arquivamento:

```sql
ALTER SYSTEM SWITCH LOGFILE;

SELECT GROUP#, THREAD#, BYTES/1024/1024 AS "TAMANHO MB",
       SEQUENCE#, MEMBERS, ARCHIVED, STATUS FROM V$LOG;
```

```
GROUP#  SEQUENCE#  ARC  STATUS
------  ---------  ---  ----------
     1          1  YES  INACTIVE
     2          2  YES  INACTIVE
     3          3  YES  ACTIVE     ← arquivado, aguardando ficar INACTIVE
    10          4  NO   CURRENT
```

> <img width="910" height="226" alt="image" src="https://github.com/user-attachments/assets/0fc011ba-a8ad-4516-8fc8-dce9106d48ff" />

---

### 3.3 Configurando múltiplos destinos de arquivamento (multiplexação)

```sql
SHOW PARAMETER LOG_ARCHIVE_DEST_;
-- Todos os 31 destinos disponíveis, nenhum configurado ainda
```

Criando um segundo destino local e configurando dois destinos adicionais:

```sql
!mkdir /home/oracle/arch

ALTER SYSTEM SET log_archive_dest_1 = 'LOCATION=/u02/oradata';
ALTER SYSTEM SET log_archive_dest_2 = 'LOCATION=/home/oracle/arch';

SHOW PARAMETER DB_RECOVERY_FILE_DEST;
-- db_recovery_file_dest = /u02/FRA, size = 5G

ALTER SYSTEM SET log_archive_dest_3 = 'LOCATION=USE_DB_RECOVERY_FILE_DEST';
```

> <img width="664" height="575" alt="image" src="https://github.com/user-attachments/assets/a4d3c77a-391b-4986-a1ba-fbef4da583bd" />
> <img width="683" height="257" alt="image" src="https://github.com/user-attachments/assets/0c435a27-4983-4a36-92fd-65d52580f52b" />


Gerando mais atividade e forçando o switch para multiplexar:

```sql
INSERT INTO TESTE VALUES(1);
/
/
/
/
/
COMMIT;

ALTER SYSTEM SWITCH LOGFILE;
```

Confirmando a multiplexação nos três destinos:

```bash
!ls -lh /home/oracle/arch
-- 1_4_1237477629.dbf  (15K)

!ls -lh /u02/oradata
-- 1_4_1237477629.dbf  (15K) — mesmo arquivo, mesmo nome

!ls -lh /u02/FRA/ORCL/archivelog/2026_07_05/
-- o1_mf_1_3_o4p14pdt_.arc  (142M)
-- o1_mf_1_4_o4p1d4rr_.arc  (15K)
```

> 💡 Note a diferença de nomenclatura: em `/u02/oradata` e `/home/oracle/arch` o nome segue o padrão default (`1_4_1237477629.dbf`), enquanto na FRA (`/u02/FRA`) o Oracle usa nomenclatura OMF própria (`o1_mf_1_4_..._.arc`) — confirma o que a aula ensina: **arquivos na FRA não seguem o padrão do LOG_ARCHIVE_FORMAT**.

---

### 3.4 Personalizando o formato de nomenclatura — LOG_ARCHIVE_FORMAT

Verificando se o parâmetro é estático ou dinâmico antes de alterar:

```sql
SELECT ISINSTANCE_MODIFIABLE FROM V$PARAMETER WHERE NAME = 'log_archive_format';
-- FALSE → estático, exige SCOPE=SPFILE + restart
```

Primeira alteração — usando `%t` (thread, minúsculo, sem padding):

```sql
ALTER SYSTEM SET log_archive_format = 'ARCH_%t_%S_%r.arc' SCOPE = SPFILE;

SHUTDOWN IMMEDIATE;
STARTUP;
```

Gerando novo archived log para observar o formato:

```sql
INSERT INTO TESTE VALUES(1);
/ / / / /
COMMIT;

ALTER SYSTEM SWITCH LOGFILE;
```

```bash
!ls -lh /home/oracle/arch
-- ARCH_1_0000000005_1237477629.arc  ← %t=1, %S=0000000005 (padded), %r=1237477629
```

Segunda alteração — trocando para `%T` (thread maiúsculo, também com padding):

```sql
ALTER SYSTEM SET log_archive_format = 'ARCH_%T_%S_%r.arc' SCOPE = SPFILE;

SHUTDOWN IMMEDIATE;
STARTUP;

INSERT INTO TESTE VALUES(1);
/ / / / /
COMMIT;

ALTER SYSTEM SWITCH LOGFILE;
```

```bash
!ls -lh /home/oracle/arch
-- ARCH_0001_0000000006_1237477629.arc  ← %T agora também com padding (0001)
```

---

## 4. Conceitos consolidados

**Sequência para habilitar ARCHIVELOG:**

```
1. SHUTDOWN IMMEDIATE
2. STARTUP MOUNT
3. ALTER DATABASE ARCHIVELOG
4. ALTER DATABASE OPEN
5. Backup FULL imediato (backups anteriores ficam inválidos)
```

**Multiplexação de archived logs:**

```
LOG_ARCHIVE_DEST_1 = destino A  →  cópia idêntica
LOG_ARCHIVE_DEST_2 = destino B  →  cópia idêntica
LOG_ARCHIVE_DEST_3 = FRA        →  cópia idêntica (nomenclatura OMF própria)

Se um destino falhar, os demais garantem que o archived log existe em algum lugar.
```

**Variáveis de LOG_ARCHIVE_FORMAT — minúscula vs maiúscula:**

```
%t / %s  → sem padding    (ex: "1", "5")
%T / %S  → com padding    (ex: "0001", "0000000005")
```

---

