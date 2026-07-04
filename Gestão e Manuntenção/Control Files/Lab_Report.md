# LAB_REPORT — Lab 03: Gerenciando Control Files

**Data de execução:** 26/06/2026  
**Ambiente:** Oracle Database 19c Enterprise Edition — Oracle Linux 9,7  

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

> 📸 **Print 03** — `03-backup-controlfile-binario-confirmacao.png`  
> Capturar: o comando executado com `Database altered.` e, em seguida, `!ls -lh /home/oracle/control_file.bkp` confirmando o arquivo físico criado no SO.

Este arquivo é uma cópia exata do control file binário. Em caso de perda, basta copiar para o local correto e reiniciar.

**Método 2 — Backup em trace (script SQL de recriação):**

```sql
-- ERRO cometido: sintaxe incorreta
ALTER DATABASE BACKUP TO TRACE;
-- ORA-00905: missing keyword

-- Sintaxe correta:
ALTER DATABASE BACKUP CONTROLFILE TO TRACE;
-- Database altered.
```

> 📸 **Print 04** — `04-ora-00905-sintaxe-incorreta-e-correcao.png`  
> Capturar: o erro `ORA-00905` seguido da execução correta com `CONTROLFILE`. Mostra o erro real cometido e a correção — mais honesto do que omitir a tentativa errada.

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
-- ORA-00210: cannot open the specified control file
-- ORA-00202: control file: '/u02/oradata/ORCL/control01.ctl'
-- ORA-27041: unable to open file
-- Linux-x86_64 Error: 2: No such file or directory
```

> 📸 **Print 05** — `05-ora-00210-control-file-ausente.png`  
> Capturar: o erro completo `ORA-00210` / `ORA-00202` / `ORA-27041`. Este é o erro que qualquer DBA reconhece imediatamente em produção — vale destacar bem no portfólio.

Com `ORA-00210` ativo, `SHUTDOWN IMMEDIATE` também falha — a instância está inoperável:

```sql
SHUTDOWN IMMEDIATE;
-- ORA-00210: cannot open the specified control file (mesmo erro)

-- Único shutdown possível neste estado:
SHUTDOWN ABORT;
-- ORACLE instance shut down.
```

Tentativa de startup falha — Oracle não encontra o control file:

```sql
STARTUP;
-- ORA-00205: error in identifying control file, check alert log for more info
```

> 📸 **Print 06** — `06-shutdown-immediate-falha-abort-necessario.png`  
> Capturar: a sequência de três tentativas — `SHUTDOWN IMMEDIATE` falhando com o mesmo `ORA-00210`, depois `SHUTDOWN ABORT` funcionando, e por fim `STARTUP` retornando `ORA-00205`. Esse print conta a história completa do diagnóstico em uma única captura.

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

> 📸 **Print 07** — `07-recuperacao-mount-open-sucesso.png`  
> Capturar: a sequência `ALTER DATABASE MOUNT` → `ALTER DATABASE OPEN` → restart completo com `Database mounted` / `Database opened`. Par direto com o Print 06 — mostra a resolução do incidente.

> ✅ Banco recuperado com todos os control files sincronizados. Tempo de resolução: ~5 minutos.

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

> 📸 **Print 08** — `08-startup-nomount-ora-00205-perda-total.png`  
> Capturar: a saída do `STARTUP` parando em `ORACLE instance started` seguido do `ORA-00205`. Este print marca o início do pior cenário do lab — vale destacar no README como o "antes" da recuperação mais complexa.

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

> 📸 **Print 09** — `09-alert-log-localizando-arquivo-trace.png`  
> Capturar: o Alert Log aberto mostrando a referência ao arquivo `.trc` gerado pelo `BACKUP CONTROLFILE TO TRACE`. Demonstra a habilidade de navegar pelos arquivos de diagnóstico do Oracle — frequentemente esquecida em portfólios júnior.

```sql
-- Recriar o control file com todos os datafiles e redo logs do ambiente
CREATE CONTROLFILE REUSE DATABASE "ORCL" RESETLOGS NOARCHIVELOG
    MAXLOGFILES 16
    MAXLOGMEMBERS 3
    MAXDATAFILES 1024
    MAXINSTANCES 8
    MAXLOGHISTORY 292
LOGFILE
  GROUP 1 '/u02/oradata/ORCL/redo01.log'  SIZE 200M BLOCKSIZE 512,
  GROUP 2 '/u02/oradata/ORCL/redo02.log'  SIZE 200M BLOCKSIZE 512,
  GROUP 3 '/u02/oradata/ORCL/redo03.log'  SIZE 200M BLOCKSIZE 512,
  GROUP 4 '/u02/oradata/ORCL/onlinelog/o1_mf_4_o3p4fcdn_.log'  SIZE 100M BLOCKSIZE 512,
  GROUP 5 (
    '/home/oracle/oradata/log1c.log',
    '/u02/oradata/log2c.log'
  ) SIZE 50M BLOCKSIZE 512,
  GROUP 6 (
    '/home/oracle/oradata/ORCL/onlinelog/o1_mf_6_o3p4p0q6_.log',
    '/u02/oradata/ORCL/onlinelog/o1_mf_6_o3p4p0s4_.log'
  ) SIZE 100M BLOCKSIZE 512
DATAFILE
  '/u02/oradata/ORCL/system01.dbf',
  '/u02/oradata/ORCL/sysaux01.dbf',
  '/u02/oradata/ORCL/undotbs01.dbf',
  '/u02/oradata/ORCL/pdbseed/system01.dbf',
  '/u02/oradata/ORCL/pdbseed/sysaux01.dbf',
  '/u02/oradata/ORCL/users01.dbf',
  '/u02/oradata/ORCL/pdbseed/undotbs01.dbf',
  '/u02/oradata/ORCL/orclpdb/system01.dbf',
  '/u02/oradata/ORCL/orclpdb/sysaux01.dbf',
  '/u02/oradata/ORCL/orclpdb/undotbs01.dbf',
  '/u02/oradata/ORCL/orclpdb/users01.dbf',
  '/u02/oradata/my_datafile1.dbf',
  '/u02/oradata/my_datafile2.dbf',
  '/u02/oradata/ORCL/datafile/o1_mf_teste_o3p063jl_.dbf',
  '/u02/oradata/ORCL/datafile/o1_mf_teste_o3p06lwy_.dbf',
  '/u02/oradata/ORCL/datafile/o1_mf_teste2_o3p07ovx_.dbf'
CHARACTER SET AL32UTF8;
-- Control file created.
```

> 📸 **Print 10** — `10-create-controlfile-control-file-created.png`  
> Capturar: a execução completa do `CREATE CONTROLFILE` terminando em `Control file created.`. Este é o print mais técnico e impressionante do lab inteiro — poucos DBAs júnior conseguem mostrar essa operação documentada com sucesso.

> 💡 **Por que todos os datafiles precisam estar listados?** O `CREATE CONTROLFILE` reconstrói o mapa completo da estrutura física. Qualquer datafile omitido ficará invisível para o Oracle — o tablespace correspondente ficará OFFLINE ou inacessível após a abertura.

---

### 4.3 Abrindo o banco após CREATE CONTROLFILE

```sql
-- Tentativa sem a opção obrigatória
ALTER DATABASE OPEN;
-- ORA-01589: must use RESETLOGS or NORESETLOGS option for database open

-- RESETLOGS é obrigatório após CREATE CONTROLFILE
ALTER DATABASE OPEN RESETLOGS;
-- Database altered.
```

> 📸 **Print 11** — `11-ora-01589-resetlogs-obrigatorio.png`  
> Capturar: o erro `ORA-01589` seguido do `ALTER DATABASE OPEN RESETLOGS` bem-sucedido. Mostra que você entende por que o RESETLOGS é exigido nesse contexto específico.

O PDB não abre automaticamente após RESETLOGS:

```sql
SHOW PDBS;
-- ORCLPDB: MOUNTED (não abriu)

ALTER PLUGGABLE DATABASE ALL OPEN;
-- Pluggable database altered.

SHOW PDBS;
-- ORCLPDB: READ WRITE NO  ✅
```

> 📸 **Print 12** — `12-pdb-mounted-para-read-write.png`  
> Capturar: os dois resultados do `SHOW PDBS` — antes (`MOUNTED`) e depois (`READ WRITE`) do `ALTER PLUGGABLE DATABASE ALL OPEN`. Par antes/depois claro para quem revisar o lab.

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

-- Restaurar tempfile do tablespace criado nos labs anteriores
ALTER TABLESPACE TEMPTBS_1 ADD TEMPFILE
    '/u02/oradata/ORCL/datafile/o1_mf_temptbs__o3p0bkvy_.tmp'
    SIZE 104857600 REUSE AUTOEXTEND ON NEXT 104857600 MAXSIZE 32767M;

-- Confirmar tempfiles restaurados
SELECT FILE_NAME FROM DBA_TEMP_FILES;
```

```
FILE_NAME
--------------------------------------------------------
/u02/oradata/ORCL/datafile/o1_mf_temptbs__o3p0bkvy_.tmp
/u02/oradata/ORCL/temp01.dbf
```

> 📸 **Print 13** — `13-dba-temp-files-restaurados.png`  
> Capturar: o resultado do `SELECT FILE_NAME FROM DBA_TEMP_FILES` confirmando os dois tempfiles restaurados. Esta é a etapa que mais DBAs júnior esquecem — vale destacar como evidência de atenção ao detalhe.

Reiniciar para confirmar que o ambiente subiu limpo após toda a recuperação:

```sql
SHUTDOWN IMMEDIATE;
STARTUP;
ALTER PLUGGABLE DATABASE ALL OPEN;
-- ORCLPDB: READ WRITE NO  ✅
```

> 📸 **Print 14** — `14-restart-final-ambiente-limpo.png`  
> Capturar: o restart completo final com `Database mounted` / `Database opened` e `SHOW PDBS` confirmando `READ WRITE`. Este é o print de fechamento do lab — prova que toda a recuperação foi bem-sucedida e o ambiente está estável.

> ✅ Recuperação completa concluída. O banco foi recriado a partir do zero usando o script de trace — sem nenhum backup binário disponível.

---

## 5. Erros encontrados e como resolvi

### Erro 1 — SHOQ PARAMETER / sqkplus (erros de digitação)

```
SP2-0734: unknown command beginning "SHOQ PARAM..."
-bash: sqkplus: command not found
```

**Causa:** erros de digitação no início da sessão.  
**Solução:** redigitar os comandos corretamente.  
**Aprendizado:** erros de digitação acontecem — documentá-los é mais honesto do que omiti-los.

---

### Erro 2 — ORA-01034 ao executar SHOW PARAMETER com instância parada

```
ORA-01034: ORACLE not available
```

**Causa:** a instância estava parada (`Connected to an idle instance`). Qualquer comando que acesse o dicionário falha.  
**Solução:** `STARTUP` antes de continuar.  
**Aprendizado:** ao conectar com `sqlplus / as sysdba`, verificar sempre o estado com `SELECT STATUS FROM V$INSTANCE` ou observar a mensagem de conexão.

---

### Erro 3 — ORA-00905 no backup em trace

```
ALTER DATABASE BACKUP TO TRACE;
ORA-00905: missing keyword
```

**Causa:** a palavra `CONTROLFILE` é obrigatória na sintaxe.  
**Solução:** `ALTER DATABASE BACKUP CONTROLFILE TO TRACE;`  
**Aprendizado:** a sintaxe completa é `ALTER DATABASE BACKUP CONTROLFILE TO { 'arquivo' | TRACE }`.

---

### Erro 4 — ORA-01119 ao criar tablespace sem especificar SIZE

```
CREATE TABLESPACE TESTE_CTL DATAFILE '/u02/oradata/my_datafile5.dbf';
ORA-01119: error in creating database file
ORA-17610: file does not exist and no size specified
```

**Causa:** quando o arquivo não existe, o Oracle precisa criá-lo — e para isso exige o `SIZE`.  
**Solução:** `CREATE TABLESPACE TESTE_CTL DATAFILE '/caminho/arquivo.dbf' SIZE 100M;`  
**Aprendizado:** `SIZE` é opcional apenas quando o arquivo já existe em disco (com `REUSE`).

---

### Erro 5 — ORA-00210 e SHUTDOWN IMMEDIATE inoperável

```
ORA-00210: cannot open the specified control file
ORA-00202: control file: '/u02/oradata/ORCL/control01.ctl'
```

**Causa:** control file removido com banco UP. Qualquer operação que grava no control file falha.  
**Solução:** `SHUTDOWN ABORT` (único modo disponível) → restaurar arquivo → `STARTUP`.  
**Aprendizado:** quando um control file fica indisponível com o banco UP, `SHUTDOWN ABORT` é o único caminho. O `SHUTDOWN IMMEDIATE` e `NORMAL` também falham porque tentam gravar no control file.

---

### Erro 6 — ORA-01589 após CREATE CONTROLFILE

```
ALTER DATABASE OPEN;
ORA-01589: must use RESETLOGS or NORESETLOGS option for database open
```

**Causa:** após `CREATE CONTROLFILE`, o Oracle exige que a opção de abertura seja declarada explicitamente.  
**Solução:** `ALTER DATABASE OPEN RESETLOGS;`  
**Aprendizado:** `RESETLOGS` reinicia a sequência de log — sempre faça backup completo do banco imediatamente após um `OPEN RESETLOGS`.

---

### Erro 7 — mv com aviso de SELinux

```bash
mv /u02/oradata/ORCL/control02.ctl /home/oracle/ct_copies/control02.ctl
mv: setting attribute 'security.selinux' for 'security.selinux': Operation not permitted
```

**Causa:** o SELinux não permite mover atributos de contexto de segurança entre sistemas de arquivo com políticas diferentes.  
**Resultado:** o arquivo foi movido com sucesso — o aviso é sobre o atributo de segurança, não sobre o arquivo em si.  
**Aprendizado:** em ambientes com SELinux ativo, `mv` entre filesystems diferentes pode gerar esse aviso. Verificar se o arquivo chegou ao destino com `ls`.

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

Data planejada: conforme calendário 1z0-082
