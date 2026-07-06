# LAB_REPORT — Lab 03: Gerenciando Control Files

**Data de execução:** 26/06/2026  
**Ambiente:** Oracle Database 19c Enterprise Edition — Oracle Linux 7.9 

---

## 1. Contexto e objetivo

O control file é o arquivo mais crítico da arquitetura Oracle. Ele registra a estrutura física completa do banco — localização de datafiles e redo logs, checkpoint SCN, sequência de log — e é lido obrigatoriamente durante a fase de MOUNT no startup. Se o Oracle não consegue abrir nenhum dos control files listados no parâmetro `CONTROL_FILES`, a instância não monta.

Este lab cobre o ciclo completo de gerenciamento: multiplexação, backup e dois cenários reais de recuperação — do mais simples (cópia binária) ao mais crítico (recriação com `CREATE CONTROLFILE` quando todos os control files são perdidos).

**Por que importa na prática:**  
Em um ambiente de produção, o DBA recebe chamados como "banco não sobe — ORA-00205" diretamente relacionados a falha de control file. Saber diagnosticar e recuperar em menos de 15 minutos é habilidade obrigatória para qualquer DBA júnior.

---

## 2. Conceitos fundamentais

### O que o control file registra

| Informação | Detalhe |
|---|---|
| Nome do banco | `db_name` — usado para identificar a instância |
| Localização dos datafiles | Todos os `.dbf` do banco, incluindo PDB |
| Localização dos redo logs | Groups e members |
| Timestamp de criação | Identidade única do banco |
| Checkpoint SCN | Ponto de consistência para recovery |
| Sequência de log atual | Controla o LGWR |

### Comportamento com múltiplos control files

| Operação | Comportamento |
|---|---|
| Gravação | Oracle grava **em todos** os control files simultaneamente |
| Leitura | Oracle lê **apenas o primeiro** listado no parâmetro `CONTROL_FILES` |
| Falha de um arquivo | Instância para imediatamente — `ORA-00210` |

### As duas formas de backup

| Método | Resultado | Quando usar |
|---|---|---|
| `BACKUP CONTROLFILE TO '/caminho/arquivo.bkp'` | Cópia binária — restauração direta por `cp` | Backup pré-manutenção e recuperação rápida |
| `BACKUP CONTROLFILE TO TRACE` | Script SQL de recriação — salvo no diretório de trace | Quando não há backup binário disponível |

---

## 3. Sequência de execução — Sessão 1

### 3.1 Estado inicial e configuração do ambiente

```bash
# Configurar variáveis de ambiente Oracle
. oraenv
ORACLE_SID = [oracle] ? orcl

-- Verificar estado dos control files
SHOW PARAMETER CONTROL;
```

Estado inicial do parâmetro:
```
NAME           VALUE
-------------- -------------------------------------------------
control_files  /u02/oradata/ORCL/control01.ctl,
               /u02/oradata/ORCL/control02.ctl
```

> <img width="667" height="146" alt="image" src="https://github.com/user-attachments/assets/b835d970-2ca1-48cb-a1ed-a34ea7978867" />

O banco já estava com 2 control files. O recomendável é no mínimo 2, idealmente 3 em discos físicos diferentes.

---

### 3.2 Adicionando um terceiro control file (multiplexação)

Antes de alterar o parâmetro `CONTROL_FILES`, criei um PFILE de backup — boa prática antes de qualquer alteração de parâmetro estático:

```sql
-- Backup do SPFILE atual antes da alteração
CREATE PFILE = '/home/oracle/pfile_before_control_file_add.ora' FROM SPFILE;
```

O parâmetro `CONTROL_FILES` é estático — a alteração exige restart:

```sql
-- Adicionar o terceiro control file no SPFILE
ALTER SYSTEM SET CONTROL_FILES =
    '/u02/oradata/ORCL/control01.ctl',
    '/u02/oradata/ORCL/control02.ctl',
    '/home/oracle/control_file.ctl'
SCOPE = SPFILE;

-- Encerrar o banco
SHUTDOWN IMMEDIATE;
```

O arquivo físico precisa ser criado **antes** do startup — o Oracle não cria o arquivo, apenas registra os que existem:

```bash
-- Copiar um control file existente para o novo caminho
!cp /u02/oradata/ORCL/control01.ctl /home/oracle/control_file.ctl
```

```sql
-- Iniciar — Oracle abre os três control files na fase MOUNT
STARTUP;

-- Confirmar que os três estão ativos
SHOW PARAMETER CONTROL;
```

```
NAME           VALUE
-------------- -------------------------------------------------
control_files  /u02/oradata/ORCL/control01.ctl,
               /u02/oradata/ORCL/control02.ctl,
               /home/oracle/control_file.ctl
```

> <img width="643" height="155" alt="image" src="https://github.com/user-attachments/assets/03d1df03-3f89-4a39-b25c-7d2c829a6510" />

>  Multiplexação concluída — banco agora mantém 3 cópias sincronizadas do control file.

---

### 3.3 Backup do control file

**Método 1 — Cópia binária:**

```sql
ALTER DATABASE BACKUP CONTROLFILE TO '/home/oracle/control_file.bkp';
-- Database altered.
```

> <img width="670" height="133" alt="image" src="https://github.com/user-attachments/assets/44e4b01a-dc55-43c7-bd7e-04f83c191835" />


Este arquivo é uma cópia exata do control file binário. Em caso de perda, basta copiar para o local correto e reiniciar.

**Método 2 — Backup em trace (script SQL de recriação):**

```sql
ALTER DATABASE BACKUP CONTROLFILE TO TRACE;
-- Database altered.
```

> <img width="818" height="78" alt="image" src="https://github.com/user-attachments/assets/dfe3c23f-1144-487c-b2f5-69675333ee84" />



O Oracle salva o script de recriação no diretório de trace (`$ORACLE_BASE/diag/rdbms/orcl/orcl/trace/`). O nome exato do arquivo aparece no Alert Log.

> 💡 O backup em trace gera um arquivo `.trc` com o comando `CREATE CONTROLFILE` completo, listando todos os datafiles e redo logs do banco no momento do backup. Esse script é a base para o Cenário B de recuperação.

---

### 3.4 Cenário A — Recuperação de um control file por cópia binária

Simulei a perda do `control01.ctl` com o banco ainda ativo:

```bash
# Remover control01.ctl com banco UP (simulação de falha de disco)
# O banco ainda não detectou a falha neste momento
```

Ao tentar executar qualquer DDL, o Oracle detectou a ausência do arquivo:

```sql
CREATE TABLESPACE TESTE_CTL DATAFILE '/u02/oradata/my_datafile5.dbf';
-- ORA-01119: error in creating database file '/u02/oradata/my_datafile5.dbf'
-- ORA-17610: file '/u02/oradata/my_datafile5.dbf' does not exist and no size specified
-- ORA-27037: unable to obtain file status
-- Linux-x86_64 Error: 2: No such file or directory
-- Additional information: 7

```

> <img width="619" height="174" alt="image" src="https://github.com/user-attachments/assets/57601756-675f-4788-9ad2-d0ff443efb96" />


Com `ORA-00210` ativo, `SHUTDOWN IMMEDIATE` também falha — a instância está inoperável:

```sql
SHUTDOWN IMMEDIATE;
-- ORA-00210: cannot open the specified control file
-- ORA-00202: control file: '/u02/oradata/ORCL/control01.ctl'
-- ORA-27041: unable to open file
-- Linux-x86_64 Error: 2: No such file or directory

-- Único shutdown possível neste estado:
SHUTDOWN ABORT;
-- ORACLE instance shut down.
```

Tentativa de startup falha — Oracle não encontra o control file:

```sql
STARTUP;
-- ORA-00205: error in identifying control file, check alert log for more info
```

> <img width="637" height="331" alt="image" src="https://github.com/user-attachments/assets/2279be81-933c-492a-a714-e5227f25d3a3" />


Recuperação por cópia binária — o backup feito anteriormente resolve em segundos:

```bash
-- Copiar o backup binário para o local original
!cp /home/oracle/control_file.bkp /u02/oradata/ORCL/control01.ctl
```

```sql
-- O banco ainda está em NOMOUNT (instância ativa sem montar)
-- Montar manualmente para verificar antes de abrir
ALTER DATABASE MOUNT;

-- Sincronizar control02 a partir do control01 restaurado
!cp /u02/oradata/ORCL/control01.ctl /u02/oradata/ORCL/control02.ctl

-- Abrir o banco
ALTER DATABASE OPEN;

-- Reiniciar completamente para confirmar que tudo subiu limpo
SHUTDOWN IMMEDIATE;
STARTUP;
-- Database mounted / Database opened
```

> <img width="526" height="163" alt="image" src="https://github.com/user-attachments/assets/9c25f316-2ab5-4bae-becf-652ff831f84b" />


> Banco recuperado com todos os control files sincronizados. 

---

## 4. Sequência de execução — Sessão 2 (cenário mais crítico)

### 4.1 Perda total de todos os control files

Simulei o pior cenário: todos os três control files removidos simultaneamente, com o banco parado:

```bash
mkdir /home/oracle/ct_copies

# Mover todos os control files para fora do ambiente
mv /u02/oradata/ORCL/control01.ctl /home/oracle/ct_copies/control01.ctl
mv /u02/oradata/ORCL/control02.ctl /home/oracle/ct_copies/control02.ctl
mv /home/oracle/control_file.ctl   /home/oracle/ct_copies/control_file.ctl
```

Ao tentar iniciar o banco sem nenhum control file:

```sql
STARTUP;
-- ORACLE instance started (fase NOMOUNT — instância ativa)
-- ORA-00205: error in identifying control file, check alert log for more info
-- (banco parou na fase MOUNT — não conseguiu abrir nenhum control file)
```

> <img width="619" height="188" alt="image" src="https://github.com/user-attachments/assets/55a564d1-fb4f-410e-a4d1-4bd7441f70ae" />


> 💡 Neste momento o banco está em estado **NOMOUNT** — a instância está ativa (SGA alocada, processos de background rodando), mas o banco não está montado. Isso é suficiente para executar `CREATE CONTROLFILE`.

---

### 4.2 Recuperação por CREATE CONTROLFILE

O comando `CREATE CONTROLFILE` recria o control file do zero a partir de informações fornecidas explicitamente. As informações vieram do backup em trace gerado anteriormente (`.trc` na pasta de diagnóstico):

```bash
# Localizar o arquivo trace com o script de recriação
cd $ORACLE_BASE/diag/rdbms/orcl/orcl/trace
view alert_orcl.log   # identifica o nome do arquivo .trc gerado
view /u01/app/oracle/diag/rdbms/orcl/orcl/trace/orcl_ora_XXXX.trc
```

> <img width="556" height="39" alt="image" src="https://github.com/user-attachments/assets/4de6d4f3-f632-4526-a31b-f56e65871885" />

> <img width="857" height="619" alt="image" src="https://github.com/user-attachments/assets/00604db8-8386-4983-b0ea-54cb8f9edecb" />


```sql
-- Recriar o control file com todos os datafiles e redo logs do ambiente
 CREATE CONTROLFILE REUSE DATABASE "ORCL" NORESETLOGS  NOARCHIVELOG
    MAXLOGFILES 16
    MAXLOGMEMBERS 3
    MAXDATAFILES 1024
    MAXINSTANCES 8
    MAXLOGHISTORY 292
LOGFILE
  GROUP 1 '/u02/oradata/ORCL/redo01.log'  SIZE 200M BLOCKSIZE 512,
  GROUP 2 (
    '/u02/oradata/ORCL/redo02.log',
    '/home/oracle/onlinelogs/g2redo2.rdo'
  ) SIZE 200M BLOCKSIZE 512,
  GROUP 3 '/u02/oradata/ORCL/redo03.log'  SIZE 200M BLOCKSIZE 512,
  GROUP 10 '/u02/oradata/ORCL/g10redo2.rdo'  SIZE 200M BLOCKSIZE 512
  2    3    4    5    6    7    8    9   10   11   12   13   14   15  -- STANDBY LOGFILE
DATAFILE
  '/u02/oradata/ORCL/system01.dbf',
  '/u02/oradata/ORCL/sysaux01.dbf',
  '/u02/oradata/ORCL/undotbs01.dbf',
  '/u02/oradata/ORCL/pdbseed/system01.dbf',
  '/u02/oradata/ORCL/pdbseed/sysaux01.dbf',
  '/u02/oradata/ORCL/users01.dbf',
  '/u02/oradata/ORCL/pdbseed/undotbs01.dbf',
  '/u02/oradata/ORCL/ORCLPDB/system01.dbf',
  '/u02/oradata/ORCL/ORCLPDB/sysaux01.dbf',
  '/u02/oradata/ORCL/ORCLPDB/undotbs01.dbf',
  '/u02/oradata/ORCL/ORCLPDB/users01.dbf'
CHARACTER SET AL32UTF8 ;
-- Control file created.
```

> <img width="730" height="589" alt="image" src="https://github.com/user-attachments/assets/99b3c0b4-7cdb-404a-a49f-3774c7fa694b" />

> 💡 **Por que todos os datafiles precisam estar listados?** O `CREATE CONTROLFILE` reconstrói o mapa completo da estrutura física. Qualquer datafile omitido ficará invisível para o Oracle — o tablespace correspondente ficará OFFLINE ou inacessível após a abertura.

---

### 4.3 Abrindo o banco após CREATE CONTROLFILE

```sql
ALTER DATABASE OPEN;

ALTER PLUGGABLE DATABASE ALL OPEN;
-- Pluggable database altered.

SHOW PDBS;
-- ORCLPDB: READ WRITE NO  ✅
```

> <img width="530" height="219" alt="image" src="https://github.com/user-attachments/assets/eb4b1ad6-40d4-4e1c-87ee-e60d176bf88e" />


---

### 4.4 Restaurando tempfiles (etapa obrigatória pós-recriação)

O `CREATE CONTROLFILE` não inclui tempfiles — eles precisam ser restaurados manualmente. Sem isso, operações que usam tablespace temporário falham silenciosamente.

```sql
-- Restaurar tempfile do CDB$ROOT
ALTER TABLESPACE TEMP ADD TEMPFILE '/u02/oradata/ORCL/temp01.dbf'
    SIZE 135266304 REUSE AUTOEXTEND ON NEXT 655360 MAXSIZE 32767M;

-- Restaurar tempfile do PDB$SEED
ALTER SESSION SET CONTAINER = "PDB$SEED";
ALTER TABLESPACE TEMP ADD TEMPFILE
    '/u02/oradata/ORCL/pdbseed/temp012026-05-28_18-05-58-021-PM.dbf'
    SIZE 37748736 REUSE AUTOEXTEND ON NEXT 655360 MAXSIZE 32767M;

-- Restaurar tempfile do ORCLPDB
ALTER SESSION SET CONTAINER = "ORCLPDB";
ALTER TABLESPACE TEMP ADD TEMPFILE '/u02/oradata/ORCL/orclpdb/temp01.dbf' REUSE;

-- Voltar ao CDB$ROOT
ALTER SESSION SET CONTAINER = "CDB$ROOT";


-- Confirmar tempfiles restaurados
SELECT FILE_NAME FROM DBA_TEMP_FILES;
```

```
FILE_NAME
--------------------------------------------------------
/u02/oradata/ORCL/temp01.dbf
```

> <img width="370" height="231" alt="image" src="https://github.com/user-attachments/assets/457c8f4d-7c86-4e6a-94ad-6ba20bfea5c8" />


Reiniciar para confirmar que o ambiente subiu limpo após toda a recuperação:

```sql
SHUTDOWN IMMEDIATE;
STARTUP;
ALTER PLUGGABLE DATABASE ALL OPEN;
-- ORCLPDB: READ WRITE NO  ✅
```

> <img width="532" height="165" alt="image" src="https://github.com/user-attachments/assets/b1768803-42a5-46e3-9f3a-d305686ddcd8" />

> Recuperação completa concluída. O banco foi recriado a partir do zero usando o script de trace — sem nenhum backup binário disponível.

---

## 5. Erros encontrados e como resolvi

---

### Erro 1 — ORA-01119 ao criar tablespace sem especificar SIZE

```
CREATE TABLESPACE TESTE_CTL DATAFILE '/u02/oradata/my_datafile5.dbf';
ORA-01119: error in creating database file
ORA-17610: file does not exist and no size specified
```

**Causa:** quando o arquivo não existe, o Oracle precisa criá-lo — e para isso exige o `SIZE`.  
**Solução:** `CREATE TABLESPACE TESTE_CTL DATAFILE '/caminho/arquivo.dbf' SIZE 100M;`  
**Aprendizado:** `SIZE` é opcional apenas quando o arquivo já existe em disco (com `REUSE`).

---

### Erro 2 — ORA-00210 e SHUTDOWN IMMEDIATE inoperável

```
ORA-00210: cannot open the specified control file
ORA-00202: control file: '/u02/oradata/ORCL/control01.ctl'
```

**Causa:** control file removido com banco UP. Qualquer operação que grava no control file falha.  
**Solução:** `SHUTDOWN ABORT` (único modo disponível) → restaurar arquivo → `STARTUP`.  
**Aprendizado:** quando um control file fica indisponível com o banco UP, `SHUTDOWN ABORT` é o único caminho. O `SHUTDOWN IMMEDIATE` e `NORMAL` também falham porque tentam gravar no control file.

---

### Erro 3 — ORA-01589 após CREATE CONTROLFILE

```
ALTER DATABASE OPEN;
ORA-01589: must use RESETLOGS or NORESETLOGS option for database open
```

**Causa:** após `CREATE CONTROLFILE`, o Oracle exige que a opção de abertura seja declarada explicitamente.  
**Solução:** `ALTER DATABASE OPEN RESETLOGS;`  
**Aprendizado:** `RESETLOGS` reinicia a sequência de log — sempre faça backup completo do banco imediatamente após um `OPEN RESETLOGS`.

---

## 6. Conceitos consolidados

**Árvore de decisão — recuperação de control file:**

```
Algum control file disponível?
 ├─ Sim  →  Copiar o disponível para o caminho do perdido       (Cenário A)
 │          STARTUP (banco monta com a cópia)
 └─ Não  →  Tenho backup binário (.bkp)?
             ├─ Sim  →  Copiar .bkp para todos os caminhos      (Cenário A)
             │          STARTUP
             └─ Não  →  Usar script do backup TO TRACE           (Cenário B)
                         CREATE CONTROLFILE + RESETLOGS
                         Restaurar tempfiles manualmente
                         BACKUP IMEDIATO do banco
```

**O que o `BACKUP CONTROLFILE TO TRACE` gera:**

```
Um arquivo .trc em $ORACLE_BASE/diag/rdbms/<db>/<instance>/trace/
Contém: CREATE CONTROLFILE com todos os datafiles e redo logs
Localização: identificada no Alert Log após execução do comando
```

**Por que tempfiles precisam ser restaurados manualmente:**  
O control file não registra tempfiles — apenas datafiles permanentes. Após `CREATE CONTROLFILE`, os tablespaces temporários existem no dicionário mas sem seus tempfiles físicos associados. Operações de sort, hash join e criação de índices falham silenciosamente até que os tempfiles sejam restaurados.

---

