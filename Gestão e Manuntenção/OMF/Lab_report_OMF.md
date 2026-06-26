# LAB_REPORT — Lab 02: Oracle Managed Files (OMF)

**Data de execução:** 25/06/2026  
**Ambiente:** Oracle Database 19c Enterprise Edition — Oracle Linux 9.7 

---

## 1. Contexto e objetivo

O Oracle Managed Files (OMF) é um recurso que transfere para o banco a responsabilidade de nomear e gerenciar os arquivos do sistema. Sem OMF, o DBA precisa especificar o caminho completo de cada datafile, redo log member e arquivo de backup — o que aumenta a chance de erro humano e dificulta a padronização em ambientes com muitos arquivos.

Com OMF ativo, o Oracle cria, estende e remove arquivos automaticamente seguindo convenções de nomenclatura padronizadas, simplificando as rotinas de administração sem perder o controle sobre a localização física.

**Por que importa na prática:**  
Em um ambiente de produção com múltiplos bancos e tablespaces criados frequentemente por diferentes DBAs, o OMF elimina variações de nomenclatura, previne arquivos órfãos em disco e reduz o tempo de execução de tarefas rotineiras como expansão de tablespace e adição de redo log groups.

---

## 2. Conceitos fundamentais

### Os três parâmetros OMF

| Parâmetro | Controla | Requisito para ativar |
|---|---|---|
| `db_create_file_dest` | Datafiles e tempfiles | Diretório deve existir |
| `db_create_online_log_dest_n` | Redo log members (até 5 destinos) | Diretório deve existir |
| `db_recovery_file_dest` | Fast Recovery Area (FRA) | `db_recovery_file_dest_size` deve ser definido **antes** |

### Comportamento dos arquivos criados pelo OMF

| Característica | Arquivo manual | Arquivo OMF |
|---|---|---|
| Nomenclatura | Definida pelo DBA | `o1_mf_<nome>_<tag>_` automática |
| AUTOEXTENSIBLE | NO (padrão) | YES (padrão) |
| MAXSIZE | 0 (sem limite definido) | 32767.9844 MB |
| Remoção ao dropar objeto | Arquivo permanece em disco | Oracle remove automaticamente |

### Ordem de prioridade para criação de redo logs

```
1. DB_CREATE_ONLINE_LOG_DEST_1 até _5  (se definido — cria member em cada destino)
2. DB_CREATE_FILE_DEST                 (fallback — cria member único)
3. ORA-02236                           (nenhum definido — exige caminho explícito)
```

---

## 3. Sequência de execução

### 3.1 Comportamento sem OMF — db_create_file_dest vazio

Primeiro verifiquei o estado inicial do ambiente:

```sql
SELECT FILE_NAME FROM DBA_DATA_FILES;
SHOW PARAMETER db_create_file_dest;
```

><img width="669" height="436" alt="Captura de tela 2026-06-25 230440" src="https://github.com/user-attachments/assets/363c0d0e-0cdc-4aad-8cb0-f4d62c10ffba" />

Com o parâmetro vazio, tentei criar tablespaces sem especificar o caminho do datafile:

```sql
-- ERRO ORA-02236: nome de arquivo inválido (sem OMF e sem caminho)
CREATE TABLESPACE teste DATAFILE SIZE 10M;

-- ERRO ORA-02199: cláusula DATAFILE/TEMPFILE ausente
CREATE TABLESPACE teste2;
```

Sem OMF, o caminho do datafile é obrigatório:

```sql
-- Forma obrigatória sem OMF: especificar caminho completo
CREATE TABLESPACE teste DATAFILE '/u02/oradata/my_datafile1.dbf' SIZE 10M;

-- Para adicionar datafile: ALTER TABLESPACE (não CREATE)
ALTER TABLESPACE teste ADD DATAFILE '/u02/oradata/my_datafile2.dbf' SIZE 10M;
```

Verificando os datafiles criados manualmente:

```sql
SET LINES 400
COL FILE_NAME FOR A70
SELECT FILE_NAME, BYTES/1024/1024 "TAMANHO MB", MAXBYTES/1024/1024 "MAX MB",
       AUTOEXTENSIBLE FROM DBA_DATA_FILES WHERE TABLESPACE_NAME='TESTE';
```

><img width="680" height="268" alt="Captura de tela 2026-06-25 230516" src="https://github.com/user-attachments/assets/0a4e9ae3-86cc-4323-a381-ccb2db421794" />

><img width="1004" height="198" alt="Captura de tela 2026-06-25 230601" src="https://github.com/user-attachments/assets/856c72c5-e320-4168-850f-847e9f98ec98" />



Arquivos criados sem OMF: `AUTOEXTENSIBLE = NO` e `MAX MB = 0` — sem extensão automática.

---

### 3.2 Ativando OMF com db_create_file_dest

O Oracle valida a existência do diretório no momento da configuração:

```sql
-- ERRO ORA-01262: diretório não existe
ALTER SYSTEM SET db_create_file_dest = '/home/oracle/ORCL' SCOPE = BOTH;

-- Diretório existente: funciona corretamente
ALTER SYSTEM SET db_create_file_dest = '/u02/oradata/' SCOPE = BOTH;

SHOW PARAMETER db_create_file_dest;
-- VALUE = /u02/oradata/
```

Com OMF ativo, o caminho do datafile passa a ser gerenciado pelo Oracle:

```sql
-- Adicionar datafile sem especificar caminho (tamanho explícito)
ALTER TABLESPACE teste ADD DATAFILE SIZE 10M;

-- Adicionar datafile sem especificar nem caminho nem tamanho
ALTER TABLESPACE teste ADD DATAFILE;
```

Comparativo dos quatro datafiles — manuais vs OMF:

```sql
SELECT FILE_NAME, BYTES/1024/1024 "TAMANHO MB", MAXBYTES/1024/1024 "MAX MB",
       AUTOEXTENSIBLE FROM DBA_DATA_FILES WHERE TABLESPACE_NAME='TESTE';
```

>><img width="1224" height="484" alt="Captura de tela 2026-06-25 230641" src="https://github.com/user-attachments/assets/821f73c4-0d81-4177-8594-8fce8e3cabc7" />


A diferença é clara: arquivos OMF têm `AUTOEXTENSIBLE = YES` e `MAX MB = 32767.9844`. O Oracle também gerou um datafile OMF com 100MB mesmo tendo sido solicitado `ADD DATAFILE` sem tamanho.

Criando tablespace permanente e temporário totalmente via OMF:

```sql
-- Tablespace permanente: Oracle define datafile, tamanho e caminho
CREATE TABLESPACE teste2;

-- Tablespace temporário: Oracle define tempfile, tamanho e caminho
CREATE TEMPORARY TABLESPACE temptbs_1;
```

Verificando a localização física dos arquivos criados pelo OMF:

```bash
ls /u02/oradata/ORCL/datafile/
# o1_mf_temptbs__o3p0bkvy_.tmp
# o1_mf_teste2_o3p07ovx_.dbf
# o1_mf_teste_o3p063jl_.dbf
# o1_mf_teste_o3p06lwy_.dbf
```

><img width="1228" height="387" alt="Captura de tela 2026-06-25 230740" src="https://github.com/user-attachments/assets/293f0376-12a8-4ff6-bd77-9a842ec56700" />

---

### 3.3 Redo logs com db_create_online_log_dest_n

Verificando os grupos e members existentes antes de qualquer alteração:

```sql
COL MEMBER FOR A70
SELECT GROUP#, MEMBER FROM V$LOGFILE;
SELECT GROUP#, BYTES/1024/1024 "TAMANHO MB" FROM V$LOG;
```

><img width="676" height="423" alt="Captura de tela 2026-06-25 230823" src="https://github.com/user-attachments/assets/3f834669-dbc8-412e-9d37-65a498dfe14e" />

Com `db_create_file_dest` ativo, `ADD LOGFILE` cria um group com um member no destino OMF:

```sql
ALTER DATABASE ADD LOGFILE;
SELECT GROUP#, MEMBER FROM V$LOGFILE;
SELECT GROUP#, BYTES/1024/1024 "TAMANHO MB" FROM V$LOG;
```

Grupo 4 criado em `/u02/oradata/ORCL/onlinelog/` com 100MB (tamanho padrão OMF).

Desativando `db_create_file_dest` para demonstrar o comportamento sem OMF:

```sql
ALTER SYSTEM SET db_create_file_dest = '' SCOPE = BOTH;

-- ERRO ORA-02236: sem OMF definido, caminho é obrigatório
ALTER DATABASE ADD LOGFILE;

-- Sem OMF: especificar cada member manualmente
ALTER DATABASE ADD LOGFILE (
    '/home/oracle/oradata/log1c.log',
    '/u02/oradata/log2c.log'
) SIZE 50M;
```

Grupo 5 criado com 2 members em discos diferentes — multiplexação manual.

Configurando `DB_CREATE_ONLINE_LOG_DEST_n` para multiplexação automática:

```sql
ALTER SYSTEM SET DB_CREATE_ONLINE_LOG_DEST_1 = '/home/oracle/oradata' SCOPE = BOTH;
ALTER SYSTEM SET DB_CREATE_ONLINE_LOG_DEST_2 = '/u02/oradata' SCOPE = BOTH;

SHOW PARAMETER DB_CREATE_ONLINE_LOG_DEST_;
```

Com os dois destinos definidos, `ADD LOGFILE` cria automaticamente um member em cada local:

```sql
ALTER DATABASE ADD LOGFILE;
SELECT GROUP#, MEMBER FROM V$LOGFILE;
```
><img width="587" height="281" alt="image" src="https://github.com/user-attachments/assets/9024f1a9-8759-4a79-a6ed-c12f32d3700e" />



Grupo 6 com 2 members em discos separados — multiplexação automática via OMF.



---

### 3.4 Fast Recovery Area (FRA) com db_recovery_file_dest

```sql
SHOW PARAMETER DB_RECOVERY_FILE_DEST;
-- VALUE vazio, SIZE = 0
```

Tentando configurar o destino antes do tamanho:

```sql
-- ERRO ORA-02097 → SIZE deve ser definido antes do destino
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST = '/u02/FRA' SCOPE = BOTH;
```

Sequência correta de configuração:

```sql
-- 1. Definir o tamanho ANTES do destino
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST_SIZE = 20G SCOPE = BOTH;

-- 2. Criar o diretório no SO
HOST mkdir /u02/FRA

-- 3. Definir o destino
ALTER SYSTEM SET DB_RECOVERY_FILE_DEST = '/u02/FRA' SCOPE = BOTH;

SHOW PARAMETER DB_RECOVERY_FILE_DEST;
-- db_recovery_file_dest      = /u02/FRA
-- db_recovery_file_dest_size = 20G
```

Verificando uso da FRA antes do backup:

```sql
SELECT * FROM V$RECOVERY_AREA_USAGE;
-- Todos os FILE_TYPE com NUMBER_OF_FILES = 0
```

><img width="774" height="384" alt="image" src="https://github.com/user-attachments/assets/fdf8bffa-ec38-477a-a061-d6e7937c0dcf" />


Gerando uso na FRA com backup via RMAN:

```bash
rman target /
RMAN> backup current controlfile;
```

><img width="1070" height="485" alt="image" src="https://github.com/user-attachments/assets/8cafd864-e75e-4511-b17c-e629e493b4a5" />

Verificando o uso da FRA após o backup:

```sql
SELECT * FROM V$RECOVERY_AREA_USAGE;
-- BACKUP PIECE: PERCENT_SPACE_USED = 0.18, NUMBER_OF_FILES = 2
```
><img width="795" height="235" alt="image" src="https://github.com/user-attachments/assets/69119931-be19-409c-a05f-cb7d1a3cf7ed" />


Estrutura criada automaticamente pelo RMAN na FRA:

```
/u02/FRA/ORCL/
├── autobackup/
│   └── 2026_06_23/
│       └── o1_mf_s_1236716395_o3p54vlg_.bkp   ← autobackup de control file + SPFILE
└── backupset/
    └── 2026_06_23/
        └── o1_mf_ncnnf_TAG20260623T201953_o3p54t6f_.bkp  ← backup do control file
```

---

## 4. Erros encontrados e como resolvi

### Erro 1 — ORA-02236 ao criar tablespace sem OMF e sem caminho

```
ORA-02236: invalid file name
```

**Causa:** `db_create_file_dest` estava vazio e nenhum caminho foi especificado no `CREATE TABLESPACE`.  
**Solução:** especificar caminho completo do datafile ou ativar o OMF primeiro.  
**Aprendizado:** o erro `ORA-02236` em `CREATE TABLESPACE` é sempre relacionado a OMF não configurado ou caminho inválido.

---

### Erro 2 — ORA-02180 ao usar ADD DATAFILE no CREATE TABLESPACE

```
ORA-02180: invalid option for CREATE TABLESPACE
```

**Causa:** `ADD DATAFILE` é cláusula do `ALTER TABLESPACE`, não do `CREATE TABLESPACE`.  
**Solução:** usar `ALTER TABLESPACE teste ADD DATAFILE`.  
**Aprendizado:** `CREATE TABLESPACE` define o primeiro datafile. Para adicionar mais, sempre `ALTER TABLESPACE`.

---

### Erro 3 — ORA-01262 ao configurar db_create_file_dest com diretório inexistente

```
ORA-01261: Parameter db_create_file_dest destination string cannot be translated
ORA-01262: Stat failed on a file destination directory
Linux-x86_64 Error: 2: No such file or directory
```

**Causa:** o Oracle valida a existência do diretório no momento da configuração do parâmetro.  
**Solução:** criar o diretório no SO antes de executar o `ALTER SYSTEM SET`.  
**Aprendizado:** para qualquer parâmetro de destino OMF (`db_create_file_dest`, `db_create_online_log_dest_n`, `db_recovery_file_dest`), o diretório deve existir previamente.

---

### Erro 4 — ORA-02097 ao configurar db_recovery_file_dest sem SIZE definido

```
ORA-02097: parameter cannot be modified because specified value is invalid
ORA-01261: Parameter db_recovery_file_dest destination string cannot be translated
```

**Causa:** o Oracle exige que `db_recovery_file_dest_size` seja definido antes de `db_recovery_file_dest`.  
**Solução:** definir o SIZE primeiro, depois o destino.  
**Aprendizado:** sequência obrigatória — SIZE → destino. Tentei múltiplas vezes antes de perceber que faltava criar o diretório `/u02/FRA` no SO.

---

## 5. Resultado do lab

| Atividade executada | Status |
|---|---|
| Identificar comportamento sem OMF (erros ORA-02236, ORA-02199) | ✅ |
| Criar tablespace com caminho explícito (sem OMF) | ✅ |
| Ativar `db_create_file_dest` e criar tablespace via OMF | ✅ |
| Comparar arquivos manuais vs OMF (AUTOEXTENSIBLE, MAXSIZE) | ✅ |
| Criar tablespace temporário via OMF | ✅ |
| Adicionar redo log via `db_create_file_dest` | ✅ |
| Multiplexar redo log via `DB_CREATE_ONLINE_LOG_DEST_n` | ✅ |
| Configurar FRA com `db_recovery_file_dest` | ✅ |
| Verificar uso da FRA com `V$RECOVERY_AREA_USAGE` | ✅ |
| Gerar backup na FRA via RMAN | ✅ |
| Erros encontrados e resolvidos | 4 de 4 |

---

## 6. Conceitos consolidados

**Diferença entre arquivos manuais e OMF:**

```
Manual: /u02/oradata/my_datafile1.dbf   → AUTOEXTENSIBLE=NO, MAX=0, arquivo permanece ao dropar
OMF:    /u02/oradata/ORCL/datafile/o1_mf_teste_o3p063jl_.dbf
                                         → AUTOEXTENSIBLE=YES, MAX=32GB, Oracle remove ao dropar
```

**Multiplexação automática de redo logs:**

```
Sem OMF:  ALTER DATABASE ADD LOGFILE ('/path1/redo.log', '/path2/redo.log') SIZE 200M;
Com OMF:  ALTER DATABASE ADD LOGFILE;
          → Oracle cria automaticamente um member em cada DB_CREATE_ONLINE_LOG_DEST_n
```

---




