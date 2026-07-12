# LAB_REPORT — Lab: Gerenciando Redo Logs

**Data de execução:** 01–05/07/2026  
**Ambiente:** Oracle Database 19c Enterprise Edition — Oracle Linux 7  

---

## 1. Contexto e objetivo

Os Online Redo Logs são a estrutura mais crítica para recuperação do Oracle Database. Todo bloco de dados alterado gera um registro redo que é gravado no Redo Log Buffer pela sessão de usuário, e o processo LGWR (Log Writer) persiste esses dados nos arquivos de redo em disco. Sem eles, o Oracle não consegue fazer instance recovery após uma falha.

Este lab cobre o ciclo completo de gerenciamento: consulta, criação com e sem OMF, adição de membros, movimentação de arquivos, remoção de grupos e membros, e controle da geração de redo por nível de CDB, PDB, tablespace e objeto.

**Por que importa na prática:**  
Em um ambiente corporativo, o DBA precisa saber adicionar grupos de redo quando o LGWR está esperando frequentemente por um grupo disponível, mover membros quando discos precisam ser substituídos, dropar grupos obsoletos após reorganizações, e configurar FORCE LOGGING corretamente em ambientes com Data Guard — todos cenários reais de chamado.

---

## 2. Conceitos fundamentais

### Status dos grupos de redo

| Status | O que significa | Pode dropar? |
|---|---|---|
| `CURRENT` | LGWR está gravando agora | ❌ — forçar SWITCH LOGFILE |
| `ACTIVE` | Necessário para instance recovery | ❌ — SWITCH + CHECKPOINT para INACTIVE |
| `INACTIVE` | Não necessário para recovery | ✅ |
| `UNUSED` | Recém-criado, nunca utilizado | ✅ |

### Funcionamento circular do LGWR

```
Grupo 1 (CURRENT) → Grupo 2 → Grupo 3 → volta ao Grupo 1
```

Quando um grupo enche, ocorre um **log switch** — o LGWR passa a escrever no próximo grupo disponível. O ideal é ter de 4 a 5 log switches por hora.

### Precedência de configurações de LOGGING

| CDB | PDB | Tablespace | Objeto | Gera Redo? |
|---|---|---|---|---|
| FORCE LOGGING | Ignorado | Ignorado | Ignorado | ✅ Sim |
| NO FORCE LOGGING | ENABLE FORCE LOGGING | Ignorado | Ignorado | ✅ Sim |
| NO FORCE LOGGING | ENABLE FORCE NOLOGGING | Ignorado | Ignorado | ❌ Não |
| NO FORCE LOGGING | não setado | FORCE LOGGING | Ignorado | ✅ Sim |
| NO FORCE LOGGING | não setado | NO FORCE LOGGING | LOGGING | ✅ Sim |
| NO FORCE LOGGING | não setado | NO FORCE LOGGING | NOLOGGING | ❌ Não |

---

## 3. Sequência de execução

### 3.1 Verificando o estado inicial

```sql
SELECT GROUP#, THREAD#, SEQUENCE#, MEMBERS, ARCHIVED, STATUS FROM V$LOG;

col member for a50
SELECT GROUP#, MEMBER FROM V$LOGFILE;
```

> <img width="685" height="355" alt="Captura de tela 2026-07-02 215958" src="https://github.com/user-attachments/assets/fa963f4d-4d69-49fc-a9a2-fd80a115dcc8" />


Estado inicial: 3 grupos (1, 2, 3), 1 membro cada, banco em NOARCHIVELOG.

---

### 3.2 Log switch e checkpoint

```sql
ALTER SYSTEM CHECKPOINT;
SELECT GROUP#, THREAD#, SEQUENCE#, MEMBERS, ARCHIVED, STATUS FROM V$LOG;

ALTER SYSTEM SWITCH LOGFILE;
SELECT GROUP#, THREAD#, SEQUENCE#, MEMBERS, ARCHIVED, STATUS FROM V$LOG;<img width="638" height="562" alt="Captura de tela 2026-07-02 220129" src="https://github.com/user-attachments/assets/1e564433-91c2-40ff-8d3d-c730a78a98b4" />

```

> <img width="638" height="562" alt="Captura de tela 2026-07-02 220129" src="https://github.com/user-attachments/assets/a46458e4-76bd-48b5-887b-b2ce3e24fad8" />


> 💡 Em NOARCHIVELOG os grupos passam direto de ACTIVE para INACTIVE após o checkpoint. Em ARCHIVELOG, o grupo só passa a INACTIVE após ser arquivado pelo ARCn.

---

### 3.3 Criando grupos sem OMF

```sql
ALTER DATABASE ADD LOGFILE
    ('/home/oracle/g4redo1.rdo', '/u02/oradata/ORCL/g4redo2.rdo') SIZE 200M;

ALTER DATABASE ADD LOGFILE GROUP 10
    ('/home/oracle/g10redo1.rdo', '/u02/oradata/ORCL/myredog10.rdo') SIZE 200M;

SELECT GROUP#, THREAD#, SEQUENCE#, MEMBERS, ARCHIVED, STATUS FROM V$LOG;
```

> <img width="868" height="201" alt="Captura de tela 2026-07-02 220920" src="https://github.com/user-attachments/assets/872bc7b1-f49d-4ad7-885f-dbe7ad8d92ff" />


---

### 3.4 Criando grupos com OMF

```sql
SHOW PARAMETER db_create;

ALTER SYSTEM SET db_create_online_log_dest_1 = '/home/oracle/';
ALTER SYSTEM SET db_create_online_log_dest_2 = '/u02/oradata/';

ALTER DATABASE ADD LOGFILE;                    -- tamanho padrão OMF
ALTER DATABASE ADD LOGFILE GROUP 9 SIZE 200M;  -- tamanho definido

SELECT GROUP#, MEMBER FROM V$LOGFILE;
```

---

### 3.5 Adicionando membro a grupo existente

```sql
ALTER DATABASE ADD LOGFILE MEMBER '/home/oracle/g2redo2.rdo' TO GROUP 2;

SELECT GROUP#, THREAD#, SEQUENCE#, MEMBERS, ARCHIVED, STATUS FROM V$LOG;
```

> 💡 O tamanho não é especificado — herda dos membros do grupo. Mesmo com OMF, o caminho é obrigatório para ADD LOGFILE MEMBER.

---

### 3.6 Renomeando e movendo membros

```sql
!mkdir /home/oracle/onlinelogs

SHUTDOWN IMMEDIATE;

!mv '/u02/oradata/ORCL/myredog10.rdo' '/u02/oradata/ORCL/g10redo2.rdo'
!mv '/home/oracle/g2redo2.rdo'         '/home/oracle/onlinelogs/g2redo2.rdo'
!mv '/home/oracle/g10redo1.rdo'        '/home/oracle/onlinelogs/g10redo1.rdo'
!mv '/home/oracle/g4redo1.rdo'         '/home/oracle/onlinelogs/g4redo1.rdo'

STARTUP MOUNT;

ALTER DATABASE RENAME FILE '/u02/oradata/ORCL/myredog10.rdo'
    TO '/u02/oradata/ORCL/g10redo2.rdo';
-- (renomeações restantes uma a uma)

ALTER DATABASE OPEN;

SELECT GROUP#, THREAD#, SEQUENCE#, MEMBERS, ARCHIVED, STATUS FROM V$LOG;
```

> <img width="875" height="417" alt="Captura de tela 2026-07-02 221731" src="https://github.com/user-attachments/assets/7b0c54e4-07fa-421a-80d5-189ff314240f" />

> <img width="926" height="447" alt="Captura de tela 2026-07-02 221800" src="https://github.com/user-attachments/assets/53a8fd02-a1d1-4b1b-84ff-27b1a485c1db" />


---

### 3.7 Removendo grupos

```sql
-- Tentativa com grupo CURRENT:
ALTER DATABASE DROP LOGFILE GROUP 4;
-- ORA-01623: log 4 is current log for instance orcl - cannot drop

ALTER SYSTEM SWITCH LOGFILE;

-- Tentativa com grupo ACTIVE:
ALTER DATABASE DROP LOGFILE GROUP 4;
-- ORA-01624: log 4 needed for crash recovery of instance orcl

ALTER SYSTEM CHECKPOINT;

SELECT GROUP#, STATUS FROM V$LOG;
-- Grupo 4: INACTIVE ✅

ALTER DATABASE DROP LOGFILE GROUP 4;
-- Database altered.
```

> <img width="611" height="192" alt="Captura de tela 2026-07-02 221902" src="https://github.com/user-attachments/assets/abb9e69b-3208-4e72-b432-c879dff46b8c" />

> <img width="910" height="279" alt="Captura de tela 2026-07-02 221920" src="https://github.com/user-attachments/assets/d0ec14c4-3eb6-4b22-986a-be2f60424553" />

> <img width="919" height="363" alt="Captura de tela 2026-07-02 221947" src="https://github.com/user-attachments/assets/e9b79667-3f65-4188-93ef-570251dc4fa5" />


> 💡 Arquivos SEM OMF permanecem no disco após o DROP:

> <img width="332" height="54" alt="Captura de tela 2026-07-02 222013" src="https://github.com/user-attachments/assets/74629189-1223-4427-8550-15d862ae6145" />

> <img width="1316" height="48" alt="Captura de tela 2026-07-02 222020" src="https://github.com/user-attachments/assets/f061dbc4-28a5-4836-ada3-acfcd066800d" />


```sql
-- Arquivos SEM OMF permanecem no disco após o DROP:
!ls /home/oracle/onlinelogs/
!ls /u02/oradata/ORCL/

!rm -f /home/oracle/onlinelogs/g4redo1.rdo
!rm -f /u02/oradata/ORCL/g4redo2.rdo

-- Arquivos COM OMF são removidos automaticamente:
ALTER DATABASE DROP LOGFILE GROUP 5;
!ls /u02/oradata/ORCL/onlinelog/
-- arquivo OMF do grupo 5 não existe mais ✅
```

---

### 3.8 Removendo membro

```sql
SELECT GROUP#, MEMBER FROM V$LOGFILE;

ALTER DATABASE DROP LOGFILE MEMBER '/u02/oradata/ORCL/g10redo2.rdo';

SELECT GROUP#, MEMBER FROM V$LOGFILE;
!ls /u02/oradata/ORCL/
```

---

### 3.9 Controlando a geração de Redo — FORCE LOGGING

**Nível de CDB:**

```sql
SELECT FORCE_LOGGING FROM V$DATABASE;
-- NO

ALTER DATABASE FORCE LOGGING;

SELECT FORCE_LOGGING FROM V$DATABASE;
-- YES

ALTER DATABASE NO FORCE LOGGING;

SELECT FORCE_LOGGING FROM V$DATABASE;
-- NO
```

> <img width="385" height="382" alt="Captura de tela 2026-07-05 221830" src="https://github.com/user-attachments/assets/cbc595e9-eb6d-49fb-95fc-6a68f51ba945" />


---

**Nível de tablespace:**

```sql
SELECT TABLESPACE_NAME, FORCE_LOGGING FROM DBA_TABLESPACES;
```

```
TABLESPACE_NAME  FOR
---------------  ---
SYSTEM           YES
SYSAUX           YES
UNDOTBS1         NO
TEMP             NO
USERS            NO
```

Tentativa de desabilitar FORCE LOGGING num tablespace que já estava desabilitado:

```sql
ALTER TABLESPACE USERS NO FORCE LOGGING;
-- ORA-12925: tablespace USERS is not in force logging mode
```

Habilitando corretamente antes de desabilitar:

```sql
ALTER TABLESPACE USERS FORCE LOGGING;
-- Tablespace altered.

SELECT TABLESPACE_NAME, FORCE_LOGGING FROM DBA_TABLESPACES;
-- USERS agora YES

ALTER TABLESPACE USERS NO FORCE LOGGING;
-- Tablespace altered. ✅
```

> <img width="545" height="528" alt="Captura de tela 2026-07-05 221933" src="https://github.com/user-attachments/assets/e1c6e1c5-e1c8-49ce-96e3-78186121c676" />


> 💡 `SYSTEM` e `SYSAUX` vêm com `FORCE_LOGGING = YES` por padrão — são tablespaces do dicionário de dados e não devem operar em NOLOGGING.

---

**Nível de PDB:**

```sql
col PDB_NAME for a20
SELECT PDB_NAME, LOGGING, FORCE_LOGGING, FORCE_NOLOGGING FROM DBA_PDBS;
```

```
PDB_NAME    LOGGING   FORCE_LOGGING   FORCE_NOLOGGING
----------  --------  --------------  ---------------
ORCLPDB     LOGGING   NO              NO
PDB$SEED    LOGGING   NO              NO
```

Habilitar FORCE LOGGING exige o PDB em modo RESTRICTED:

```sql
ALTER SESSION SET CONTAINER = ORCLPDB;

ALTER PLUGGABLE DATABASE orclpdb OPEN READ WRITE RESTRICTED FORCE;

SHOW PDBS;
--     CON_ID CON_NAME   OPEN MODE   RESTRICTED
--          3 ORCLPDB    READ WRITE  YES

ALTER PLUGGABLE DATABASE orclpdb ENABLE FORCE LOGGING;

SELECT PDB_NAME, LOGGING, FORCE_LOGGING, FORCE_NOLOGGING FROM DBA_PDBS;
-- ORCLPDB: FORCE_LOGGING = YES ✅

ALTER PLUGGABLE DATABASE orclpdb OPEN READ WRITE FORCE;
-- volta ao modo normal (RESTRICTED = NO), mantendo FORCE_LOGGING = YES
```

> <img width="643" height="857" alt="Captura de tela 2026-07-05 222043" src="https://github.com/user-attachments/assets/9a98d270-8436-4f29-b379-59fc160637b6" />


Desabilitar FORCE LOGGING e habilitar FORCE NOLOGGING:

```sql
ALTER PLUGGABLE DATABASE orclpdb OPEN READ WRITE RESTRICTED FORCE;
ALTER PLUGGABLE DATABASE orclpdb DISABLE FORCE LOGGING;

SELECT PDB_NAME, LOGGING, FORCE_LOGGING, FORCE_NOLOGGING FROM DBA_PDBS;
-- FORCE_LOGGING = NO

ALTER PLUGGABLE DATABASE orclpdb ENABLE FORCE NOLOGGING;
ALTER PLUGGABLE DATABASE orclpdb OPEN READ WRITE FORCE;

SELECT PDB_NAME, LOGGING, FORCE_LOGGING, FORCE_NOLOGGING FROM DBA_PDBS;
-- FORCE_NOLOGGING = YES ✅
```

Voltando ao estado inicial (nem FORCE LOGGING nem FORCE NOLOGGING):

```sql
ALTER PLUGGABLE DATABASE orclpdb OPEN READ WRITE RESTRICTED FORCE;
ALTER PLUGGABLE DATABASE orclpdb DISABLE FORCE NOLOGGING;
ALTER PLUGGABLE DATABASE orclpdb OPEN READ WRITE FORCE;
```

> <img width="652" height="338" alt="Captura de tela 2026-07-05 222054" src="https://github.com/user-attachments/assets/191bdda4-108b-4b26-873d-bf6ab980b1b5" />

---

**Nível de objeto:**

```sql
CREATE TABLE TESTE (C1 NUMBER) NOLOGGING;
-- Table created.
```

> 💡 Tabela NOLOGGING não gera redo em operações direct-path (como INSERT /*+ APPEND */). Não é possível fazer rollforward desses dados em recovery — por isso a boa prática é backup antes e depois de operações NOLOGGING.

---

## 4. Erros encontrados e como resolvi


### Erro 1 — ORA-27038: created file already exists

**Causa:** arquivo físico já existia de tentativa anterior.  
**Solução:** `rm` do arquivo no SO antes de recriar.  
**Aprendizado:** o Oracle não sobrescreve arquivos de redo existentes — verificar com `!ls` antes de recriar.

---

### Erro 2 — ORA-01516: nonexistent log file no RENAME múltiplo

**Causa:** caminho no dicionário não correspondia ao especificado (arquivo já renomeado em tentativa anterior).  
**Solução:** renomear um arquivo por vez, confirmando os caminhos no V$LOGFILE antes.  
**Aprendizado:** RENAME múltiplo é frágil — um por vez é mais seguro.

---

### Erro 3 — ORA-01623: cannot drop CURRENT log

**Causa:** tentativa de DROP no grupo com status CURRENT.  
**Solução:** `ALTER SYSTEM SWITCH LOGFILE` primeiro.  
**Aprendizado:** o grupo CURRENT nunca pode ser dropado.

---

### Erro 4 — ORA-01624: log needed for crash recovery

**Causa:** grupo em status ACTIVE após o switch.  
**Solução:** `ALTER SYSTEM CHECKPOINT` para forçar INACTIVE.  
**Aprendizado:** sequência completa para dropar: SWITCH → CHECKPOINT → confirmar INACTIVE → DROP.

---

### Erro 5 — ORA-01109: database not open

**Causa:** `SWITCH LOGFILE` com banco em MOUNT.  
**Solução:** `ALTER DATABASE OPEN` antes.  
**Aprendizado:** log switch exige banco OPEN.

---

### Erro 6 — ORA-12925: tablespace not in force logging mode

```sql
ALTER TABLESPACE USERS NO FORCE LOGGING;
-- ORA-12925: tablespace USERS is not in force logging mode
```

**Causa:** tentei desabilitar FORCE LOGGING num tablespace que nunca havia sido habilitado — o `USERS` já estava com `FORCE_LOGGING = NO` por padrão (confirmado antes no `DBA_TABLESPACES`).  
**Solução:** habilitar primeiro (`ALTER TABLESPACE USERS FORCE LOGGING`), confirmar no `DBA_TABLESPACES`, e só então desabilitar.  
**Aprendizado:** assim como o `DROP` de redo log exige checar o status antes, o `NO FORCE LOGGING` exige que o tablespace esteja de fato em force logging mode — o Oracle não trata como no-op silencioso, retorna erro explícito.

---



## 5. Conceitos consolidados

**Ciclo de vida para dropar um grupo:**

```
DROP em grupo CURRENT  → ORA-01623
  └─ SWITCH LOGFILE
DROP em grupo ACTIVE   → ORA-01624
  └─ CHECKPOINT
DROP em grupo INACTIVE → ✅ Database altered
```

**Comportamento físico no DROP:**

```
Grupo criado SEM OMF → arquivo permanece no disco → rm manual
Grupo criado COM OMF → arquivo removido automaticamente
```

**Ciclo de FORCE LOGGING em tablespace (regra do ORA-12925):**

```
NO FORCE LOGGING em tablespace já NO FORCE LOGGING → ORA-12925
  └─ Verificar estado atual no DBA_TABLESPACES antes de alterar
  └─ Habilitar (FORCE LOGGING) → confirmar YES → só então desabilitar
```

**Habilitando FORCE LOGGING em PDB (sequência obrigatória):**

```
1. OPEN READ WRITE RESTRICTED FORCE   (reabre em modo restrito)
2. ENABLE FORCE LOGGING               (a alteração em si)
3. OPEN READ WRITE FORCE              (reabre em modo normal)
```

O mesmo padrão RESTRICTED → alteração → normal se aplica a `DISABLE FORCE LOGGING`, `ENABLE FORCE NOLOGGING` e `DISABLE FORCE NOLOGGING` — toda alteração de logging no PDB precisa desse "invólucro".

---

