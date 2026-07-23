# LAB — Configurando Oracle Data Guard (19c) — Standby Físico
**Datas:** 21–22/07/2026  
**Ambiente:** 2× VM Oracle Linux 7 · Oracle Database 19c Enterprise Edition (19.3.0.0.0)  

---

## 1. Objetivo
Construir do zero um ambiente **Oracle Data Guard** com um banco **primário** (`MERC`) e um **standby físico** (`MERC02`) em duas VMs, cobrindo: pré-requisitos do primário, preparação de rede, criação do standby via **RMAN `DUPLICATE ... FROM ACTIVE DATABASE`**, configuração do **transporte de redo**, início do **Managed Recovery (MRP)** e validação da **aplicação de redo** no standby.

## 2. Topologia

| Papel | Host | IP | DB_NAME | DB_UNIQUE_NAME | Serviço TNS |
|---|---|---|---|---|---|
| **Primário** | `nerv01` | 192.168.0.121 | `MERC` | `MERC` | `MERC` |
| **Standby** | `nerv02` | 192.168.0.122 | `MERC` | `MERC02` | `MERC02` |

> Ambas são VMs **clonadas**. `DBID` do primário: `1536142525`. CDB com a PDB `MERCP` (GUID `571E95E3B2FA4BD3E0637900A8C0BD6E`).

---

## 3. Progresso

### ✅ Fase 1 — Pré-requisitos no primário (nerv01)
Já validados em sessão anterior: banco `MERC` aberto, **ARCHIVELOG** e **FORCE LOGGING** habilitados. Conferência da estrutura:
```sql
-- RMAN
REPORT SCHEMA;              -- 12 datafiles (CDB + PDB$SEED + MERCP)

-- SQL*Plus
SELECT * FROM V$LOGFILE;    -- 3 grupos de redo, multiplexados (oradata + FRA)
```
> <img width="1032" height="511" alt="image" src="https://github.com/user-attachments/assets/0ad1b55e-af1d-4e27-be90-796624dd6471" />


### ✅ Fase 2 — Rede: listener no IP real + tnsnames (nos dois nós)
O listener estava escutando em `::1` (localhost), inútil para Data Guard. Corrigido para o **IP real** de cada nó:
```bash
# nerv01 → HOST=192.168.0.121 ; nerv02 → HOST=192.168.0.122
vi $ORACLE_HOME/network/admin/listener.ora
lsnrctl stop ; lsnrctl start
# nerv01: Listening on ... (HOST=192.168.0.121)(PORT=1521)  ✅
# nerv02: Listening on ... (HOST=192.168.0.122)(PORT=1521)  ✅
```
E os aliases TNS cruzados (`MERC` → nerv01, `MERC02` → nerv02):
```bash
tnsping MERC02     # de nerv01 → OK
tnsping MERC       # de nerv02 → OK
```
> <img width="763" height="379" alt="image" src="https://github.com/user-attachments/assets/be3131b2-7d90-43bf-94aa-8d3640a388a6" />

### ✅ Fase 3 — Copiar password file e spfile para o standby
```bash
# de nerv01:
scp $ORACLE_HOME/dbs/orapwMERC   nerv02:$ORACLE_HOME/dbs/
scp $ORACLE_HOME/dbs/spfileMERC.ora nerv02:$ORACLE_HOME/dbs/
```
> O standby precisa do **mesmo password file** do primário (autenticação do redo transport / RMAN AUXILIARY) e de um spfile de partida.

### ✅ Fase 4 — Preparar o standby (nerv02) para o NOMOUNT
O `startup nomount` falhou porque os diretórios apontados no spfile (herdado do primário) não existiam:
```
ORA-01261: Parameter db_recovery_file_dest destination string cannot be translated
ORA-01262: Stat failed on a file destination directory
```
Descobri os caminhos exatos lendo o spfile e criei a árvore:
```bash
strings $ORACLE_HOME/dbs/spfileMERC.ora | grep /
mkdir -p /u01/app/oracle/admin/MERC/adump
mkdir -p /u01/app/oracle/oradata/MERC/{controlfile,datafile,onlinelog}
mkdir -p /u01/app/oracle/oradata/MERC/571E95E3B2FA4BD3E0637900A8C0BD6E/datafile
mkdir -p /u01/app/oracle/fast_recovery_area/MERC/{controlfile,onlinelog,archivelog}

sqlplus / as sysdba
STARTUP NOMOUNT;                                   -- ✅ instância no ar
ALTER SYSTEM SET DB_UNIQUE_NAME=MERC02 SCOPE=SPFILE;
SHUTDOWN IMMEDIATE ; STARTUP NOMOUNT;              -- reinicia com o novo unique name
ALTER SYSTEM REGISTER;                             -- registra o serviço no listener
```


### ✅ Fase 5 — Criar o standby com RMAN `DUPLICATE ... FROM ACTIVE DATABASE`
Aqui foi o coração do lab — e onde apanhei bastante (ver seção de erros). A conexão do RMAN precisa do **target** (primário, montado/aberto) e do **auxiliary** (standby, em nomount):
```bash
rman TARGET SYS/oracle AUXILIARY=SYS/oracle@MERC02
# connected to target database: MERC (DBID=1536142525)
# connected to auxiliary database: MERC (not mounted)

RMAN> DUPLICATE TARGET DATABASE FOR STANDBY FROM ACTIVE DATABASE DORECOVER NOFILENAMECHECK;
```
O RMAN então, de forma automática: setou os `control_files`, restaurou o **standby controlfile**, montou o standby, copiou os **12 datafiles** para os caminhos `MERC02` (OMF), arquivou o log corrente, catalogou e aplicou os archivelogs, e rodou o `recover ... standby` até o SCN:
```
media recovery complete, elapsed time: 00:00:00
Finished Duplicate Db at 22-JUL-26
```
> <img width="918" height="505" alt="image" src="https://github.com/user-attachments/assets/c401bba7-e575-4df3-ab02-16ede50459b2" />

### ✅ Fase 6 — Configurar o transporte de redo no primário (MERC)
```sql
ALTER SYSTEM SET LOG_ARCHIVE_CONFIG='DG_CONFIG=(MERC,MERC02)';
!mkdir -p /u01/archives/MERC/
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1=
  'LOCATION=/u01/archives/MERC/ VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=MERC';
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2=
  'SERVICE=MERC02 ASYNC VALID_FOR=(ONLINE_LOGFILES,PRIMARY_ROLE) DB_UNIQUE_NAME=MERC02';
ALTER SYSTEM SET LOG_ARCHIVE_DEST_STATE_2=ENABLE SCOPE=BOTH;

-- forçar geração/envio de redo
ALTER SYSTEM SWITCH LOGFILE;
SELECT NAME, ROLE, THREAD#, SEQUENCE#, ACTION FROM V$DATAGUARD_PROCESS ORDER BY NAME, ROLE;
-- TT03 'async ORL multi' / TT04 'async ORL single' → processos de transporte de redo ativos
```
> <img width="807" height="614" alt="image" src="https://github.com/user-attachments/assets/7b2505e8-cb09-408e-b0a2-d42fb8cab0f2" />


### ✅ Fase 7 — Iniciar o Managed Recovery e validar (standby MERC02)
No standby (montado após o duplicate):
```sql
ALTER SYSTEM SET LOG_ARCHIVE_DEST_1=
  'LOCATION=/u01/archives/MERC/ VALID_FOR=(ALL_LOGFILES,ALL_ROLES) DB_UNIQUE_NAME=MERC02';
ALTER SYSTEM SET FAL_SERVER=MERC;                 -- de onde buscar gaps de archive
ALTER SYSTEM SET STANDBY_FILE_MANAGEMENT=AUTO;    -- datafiles novos replicam sozinhos
ALTER SYSTEM SET LOG_ARCHIVE_CONFIG='DG_CONFIG=(MERC,MERC02)' SCOPE=BOTH;

-- inicia o MRP (aplicação de redo em background)
ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT FROM SESSION;
```
Validação — **a prova de que o Data Guard funciona**:
```sql
-- processos de recovery/recepção
SELECT process, status, sequence# FROM v$managed_standby WHERE process IN ('MRP0','RFS');
-- MRP0  WAIT_FOR_LOG  30      (aplicando; esperando o próximo log)
-- RFS   IDLE          30      (recebendo redo do primário)

-- último log APLICADO no standby
SELECT MAX(sequence#) FROM v$archived_log WHERE applied='YES';   -- 28

-- comparar com o último gerado no primário  (rodar no primário)
SELECT thread#, MAX(sequence#) FROM v$archived_log GROUP BY thread#;   -- 28..30
```
Standby aplicando a sequência **28** enquanto o primário gera **28–30** = sincronizado, com o gap normal fechando a cada switch de log.
> <img width="658" height="247" alt="image" src="https://github.com/user-attachments/assets/a50e737d-2fa1-4183-9a80-a79bd10efd95" />
><img width="864" height="285" alt="image" src="https://github.com/user-attachments/assets/8ada34a2-7637-4487-b05b-b2b48e682852" />

><img width="940" height="597" alt="image" src="https://github.com/user-attachments/assets/4719319c-cc15-47e0-8d4c-cfebc16f498f" />
><img width="760" height="436" alt="image" src="https://github.com/user-attachments/assets/3a7c5cf8-bec1-4d53-9bfc-2bb09d4012fb" />




---

## 4. Erros encontrados e como resolvi

| Erro | Fase | Causa | Solução |
|---|---|---|---|
| `Listening on (HOST=::1)` | Rede | Listener herdado do clone escutando em localhost | Ajustar `HOST` do `listener.ora` para o **IP real** e reiniciar |
| `TNS-12545 / Linux Error 99` | Rede | `HOST` do listener não é endereço da máquina | Usar o IP/host correto do nó |
| `ORA-01017: invalid username/password` (RMAN AUXILIARY) | Duplicate | Password file do standby **não batia** com a senha do SYS | Padronizar a senha (`ALTER USER SYS IDENTIFIED BY oracle`) e **recopiar** o `orapwMERC` via `scp` |
| `RMAN-05502: target database must be mounted` | Duplicate | Primário estava só em `NOMOUNT` | Subir o primário para `MOUNT`/`OPEN` antes do duplicate |
| `ORA-19505 / ORA-17628: failed to identify file ""` | Duplicate | Diretórios do standby (controlfile/OMF) não existiam | `mkdir -p` da árvore `oradata/MERC/...` e `fast_recovery_area/MERC/...` no standby |
| `ORA-01261 / ORA-01262` | Standby nomount | `db_create_file_dest`/`db_recovery_file_dest` do spfile sem diretório físico | Criar os diretórios (`admin/MERC/adump`, `oradata`, `fast_recovery_area`) |
| `ORA-02065: illegal option for ALTER SYSTEM` | Config | Nome de parâmetro digitado errado (`LOG_ARCHIVE_1`, `SET_DBUNIQUE_NAME`, `SCOP=BOTH`) | Usar o nome exato (`LOG_ARCHIVE_DEST_1`, `DB_UNIQUE_NAME`, `SCOPE=BOTH`) |
| `ORA-03135: connection lost contact` | Config standby | Instância reiniciou/caiu durante um `ALTER SYSTEM` | Reconectar; instância voltou como `MOUNTED`, reexecutar o comando |
| `ORA-01507: database not mounted` (shutdown) | Vários | `shutdown immediate` com instância só em nomount | Normal — `shutdown abort`/reiniciar conforme o estado |
| `ORA-16047: DGID mismatch between destination setting and target database` | Config redo | `LOG_ARCHIVE_CONFIG` (`DG_CONFIG`) ainda não definido no standby, então o `DB_UNIQUE_NAME` do `LOG_ARCHIVE_DEST_2` não "casava" | Definir `LOG_ARCHIVE_CONFIG='DG_CONFIG=(MERC,MERC02)'` **nos dois lados** |
| `FAL: DGID from FAL client not in Data Guard configuration` | Transporte | Mesma raiz do ORA-16047 — standby fora da `DG_CONFIG` | Após o `LOG_ARCHIVE_CONFIG` nos dois nós, o RFS passou a receber e o MRP a aplicar |
| `ORA-00313 / ORA-27037` (online logs) no standby | Mount standby | O standby não tem os online redo logs físicos (só metadados) | Esperado — o Oracle roda `ALTER DATABASE CLEAR LOGFILE GROUP n` sozinho; não é falha |
| `Thread 1 cannot allocate new log — Checkpoint not complete` | Primário | Poucos/pequenos redo groups para o ritmo de switches | Aceitável em lab; em produção, aumentar tamanho/nº de redo logs |

---



## 5.1 Evidência dos alert logs (o que os logs comprovam)

Os alert logs dos dois nós fecham a prova ponta a ponta. Trechos-chave para citar no portfólio:

**No standby (`nerv02`) — papel e recovery:**
```
.... Database role set to PHYSICAL STANDBY
Physical Standby Database mounted.
WARNING: There are no standby redo logs.
  Standby redo logs should be configured for real time apply. Real time apply will be ignored.
Starting background process MRP0
MRP0 (PID:31600): Managed Standby Recovery not using Real Time Apply
rfs (PID:2434): Primary database is in MAXIMUM PERFORMANCE mode
MRP0 (PID:31600): Media Recovery Log /u01/archives/MERC/1_29_1239179393.dbf
MRP0 (PID:31600): Media Recovery Waiting for T-1.S-30 (in transit)   ← aplicando ao vivo
```

**No primário (`nerv01`) — o troubleshooting do transporte:**
```
ORA-16047: DGID mismatch between destination setting and target database
TT04: Error 16047 for LNO:2 to 'MERC02'
FAL: DGID from FAL client not in Data Guard configuration     (repetido)
...
ALTER SYSTEM SET log_archive_config='DG_CONFIG=(MERC,MERC02)' SCOPE=BOTH;   ← a correção
Thread 1 advanced to log sequence 30 (LGWR switch)
ARC3: Archived Log entry 39 added for T-1.S-29
```



**Data Guard operacional:** redo sendo gerado no primário, transportado (ASYNC, MAXIMUM PERFORMANCE) e **aplicado** no standby pelo MRP0, que já alcançou o primário e fica aguardando o próximo log em trânsito. 🎯

> ⚠️ **Ainda sem Standby Redo Logs (SRL):** os alert logs avisam `There are no standby redo logs ... Real time apply will be ignored`. A aplicação hoje é via archived logs (ARCH), não em tempo real. Criar SRL nos dois lados (seção 7) habilita o Real-Time Apply e reduz a perda potencial em um failover.

---

