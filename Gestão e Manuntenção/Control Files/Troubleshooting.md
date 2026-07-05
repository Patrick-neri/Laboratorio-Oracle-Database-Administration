# TROUBLESHOOTING — Incidentes reais durante os labs de Control Files
 
**Lab relacionado:** Lab 03 — Gerenciando Control Files 
**Ambiente:** Oracle Database 19c Enterprise Edition — Oracle Linux 7  
**Propósito:** registro de incidentes que ocorreram ao refazer o laboratório em sessões diferentes, com diagnóstico, resolução e causa raiz de cada um.
 
> Este documento complementa o [LAB_REPORT.md](./LAB_REPORT.md). Enquanto o lab report documenta a prática planejada, este arquivo registra os imprevistos reais — que são os cenários mais próximos do dia a dia de produção.
 
---
 
## Incidente 1 — ORA-00205: control02.ctl no caminho errado (herança da FRA)
 
**Data:** 02/07/2026  
**Contexto:** ao refazer o lab de multiplexação de control files, o banco falhou no startup após alterar o parâmetro `CONTROL_FILES` e fazer o shutdown.
 
### Sintoma
 
```sql
STARTUP;
-- ORACLE instance started.
-- ORA-00205: error in identifying control file, check alert log for more info
```
 
### Diagnóstico
 
O parâmetro apontava `control02.ctl` para `/u02/oradata/ORCL/`, mas o arquivo não estava lá:
 
```bash
cd /u02/oradata/ORCL
rm control02.ctl
# rm: cannot remove 'control02.ctl': No such file or directory
# → confirmação: o arquivo não existe neste caminho
 
cd /u02/FRA/ORCL/
ls
# control02.ctl  ← o arquivo estava AQUI
```
 
**Origem:** o `control02.ctl` foi criado em `/u02/FRA/ORCL/` durante o lab de OMF, quando o `db_recovery_file_dest` foi configurado.
 
### Resolução
 
```bash
cp /u02/FRA/ORCL/control02.ctl /u02/oradata/ORCL/control02.ctl
```
 
```sql
SHUTDOWN IMMEDIATE;
STARTUP;
-- Database mounted. / Database opened. ✅
```
 
### Aprendizado
 
- `SHOW PARAMETER` mostra a **configuração**, não a **existência** do arquivo no disco.
- Labs encadeados deixam configurações residuais — verificar o estado real do ambiente antes de assumir o estado inicial da aula.
---
 
## Incidente 2 — ORA-01152: recovery com backup control file sem archived logs
 
**Data:** 04/07/2026  
**Contexto:** após restaurar um control file de backup durante os exercícios, o banco exigiu recovery — mas os diretórios de archived logs estavam vazios.
 
### Sintoma
 
```sql
ALTER DATABASE OPEN;
-- ORA-01589: must use RESETLOGS or NORESETLOGS option for database open
 
ALTER DATABASE OPEN RESETLOGS;
-- ORA-01152: file 1 was not restored from a sufficiently old backup
-- ORA-01110: data file 1: '/u02/oradata/ORCL/system01.dbf'
```
 
O ORA-01589 era só sintoma — o problema real era o ORA-01152: o control file restaurado era um backup, e o datafile 1 (`system01.dbf`) estava "à frente" dele. O Oracle exigia aplicar o redo da **sequence #14** para tornar o banco consistente, mas o `RECOVER` anterior havia sido cancelado sem aplicar esse log.
 
### Diagnóstico
 
**Passo 1 — Verificar se existiam archived logs:**
 
```sql
HOST ls -la /u02/FRA/ORCL/archivelog/2026_07_01/
-- total 0  → vazio
 
HOST ls -la /u02/FRA/ORCL/archivelog/
-- diretórios 2026_06_30 e 2026_07_01 vazios
-- → NENHUM archived log disponível
```
 
**Passo 2 — O redo da sequence #14 ainda poderia estar nos ONLINE redo logs:**
 
```sql
STARTUP MOUNT;
 
SELECT l.group#, l.sequence#, l.status, f.member
FROM v$log l JOIN v$logfile f ON l.group# = f.group#
ORDER BY l.group#;
```
 
```
GROUP#  SEQUENCE#  STATUS    MEMBER
------  ---------  --------  --------------------------------
     1         10  INACTIVE  /u02/oradata/ORCL/redo01.log
     2         11  INACTIVE  /u02/oradata/ORCL/redo02.log
     2         11  INACTIVE  /home/oracle/onlinelogs/g2redo2.rdo
     3         14  CURRENT   /u02/oradata/ORCL/redo03.log   ← AQUI
    10         13  INACTIVE  (sem member listado — resíduo de lab)
```
 
**A sequence #14 estava no group 3 (CURRENT), member `/u02/oradata/ORCL/redo03.log`.** O redo necessário nunca saiu do disco — só não tinha sido arquivado porque o banco operava em NOARCHIVELOG.
 
### Resolução
 
No prompt do RECOVER, informar o caminho do **online redo log** no lugar do archived log:
 
```sql
RECOVER DATABASE USING BACKUP CONTROLFILE UNTIL CANCEL;
-- ORA-00279: change 2534305 generated at 07/01/2026 09:54:40 needed for thread 1
-- ORA-00289: suggestion : /u02/FRA/ORCL/archivelog/2026_07_01/o1_mf_1_14_%u_.arc
-- ORA-00280: change 2534305 for thread 1 is in sequence #14
-- Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
 
/u02/oradata/ORCL/redo01.log
-- ORA-00328: archived log ends at change 2519524, need later change 2534305
-- ORA-00334: archived log: '/u02/oradata/ORCL/redo01.log'
-- → redo01 continha a sequence 10 — log errado
```
 
Segunda tentativa, com o member correto (o que continha a sequence 14):
 
```sql
RECOVER DATABASE USING BACKUP CONTROLFILE UNTIL CANCEL;
-- Specify log: ...
 
/u02/oradata/ORCL/redo03.log
-- Log applied.
-- Media recovery complete. ✅
```
 
Abertura e confirmação:
 
```sql
ALTER DATABASE OPEN RESETLOGS;
-- Database altered.
 
ALTER PLUGGABLE DATABASE ALL OPEN;
-- Pluggable database altered.
 
SHOW PDBS;
--     CON_ID CON_NAME    OPEN MODE  RESTRICTED
--          2 PDB$SEED    READ ONLY  NO
--          3 ORCLPDB     READ WRITE NO   ✅
```
 
Backup imediato após o RESETLOGS:
 
```sql
ALTER DATABASE BACKUP CONTROLFILE TO '/home/oracle/control_file_ok.bkp' REUSE;
-- Database altered.
-- (REUSE evita o ORA-27038 caso o arquivo já exista)
```
 
### Causa raiz
 
Control file restaurado de backup + `CANCEL` respondido no primeiro RECOVER sem aplicar o redo da sequence exigida. Sem archived logs (banco em NOARCHIVELOG), a única fonte do redo era o online redo log — que felizmente ainda continha a sequence #14 no grupo CURRENT.
 
### Aprendizado
 
- Control file restaurado de backup **sempre** exige `RECOVER ... USING BACKUP CONTROLFILE` seguido de `OPEN RESETLOGS`.
- Sem archived logs, os **online redo logs** podem conter o redo necessário — o prompt `Specify log:` aceita o caminho de um member diretamente.
- Identificar qual grupo contém a sequence exigida via `V$LOG` + `V$LOGFILE` — o grupo `CURRENT` geralmente tem a última sequence gerada antes da queda.
- `ORA-00328` = log com sequence errada; tentar o próximo member até acertar.
- Após `OPEN RESETLOGS`: backup imediato — os backups e archived logs antigos ficam inválidos para recovery futuro.
---
 
## Incidente 3 — ORA-00214: versões inconsistentes de control file após mv com caminho errado
 
**Data:** 04/07/2026  
**Contexto:** ao simular a perda dos control files movendo-os para `/home/oracle/ct_copies/`, um dos comandos `mv` falhou silenciosamente — e o efeito só apareceu depois, em cascata.
 
### Sintoma (em cascata)
 
```sql
-- Movendo os 4 control files para simular perda:
!mv /u02/oradata/ORCL/control01.ctl /home/oracle/ct_copies/control01.ctl
!mv u02/oradata/ORCL/control02.ctl /home/oracle/ct_copies/control02.ctl
-- mv: cannot stat 'u02/oradata/ORCL/control02.ctl': No such file or directory
-- ← FALTOU A / INICIAL no caminho — o control02.ctl NÃO foi movido
 
!mv /home/oracle/control_file.ctl /home/oracle/ct_copies/control_file.ctl
!mv /u02/FRA/ORCL/control02.ctl /home/oracle/ct_copies/control02_fra.ctl
 
SHUTDOWN IMMEDIATE;
-- Database closed.
-- ORA-00210: cannot open the specified control file
-- ORA-00202: control file: '/u02/oradata/ORCL/control01.ctl'
 
SHUTDOWN ABORT;
STARTUP;
-- ORA-00205 (só o control02 existia nos caminhos do parâmetro)
```
 
Movendo os arquivos de volta e tentando subir:
 
```sql
SHUTDOWN ABORT;
!mv /home/oracle/ct_copies/control01.ctl /u02/oradata/ORCL/control01.ctl
!mv /home/oracle/ct_copies/control_file.ctl /home/oracle/control_file.ctl
!mv /home/oracle/ct_copies/control02_fra.ctl /u02/FRA/ORCL/control02.ctl
 
STARTUP;
-- ORACLE instance started.
-- ORA-00214: control file '/u02/oradata/ORCL/control02.ctl' version 2994
--            inconsistent with file '/u02/oradata/ORCL/control01.ctl' version 2977
```
 
### Diagnóstico
 
O `mv` do `control02.ctl` falhou por causa do caminho digitado sem a `/` inicial (`u02/...` em vez de `/u02/...`) — então ele **ficou no lugar o tempo todo**.
 
Enquanto o banco ainda rodava (e durante as tentativas de shutdown), o Oracle **continuou escrevendo nele** — por isso ele avançou para a versão 2994. As três cópias movidas pararam de ser atualizadas no momento do `mv` — congelaram na versão 2977, mais antigas. Ao trazê-las de volta, o mount comparou as versões e falhou com ORA-00214.
 
**Por que o SHUTDOWN IMMEDIATE reclamou do control01 (que tinha sido movido)?**  
No Linux, quando um arquivo aberto por um processo é movido ou removido, o processo continua escrevendo no descritor antigo sem perceber. O erro só se manifesta quando o Oracle precisa **reabrir o arquivo pelo caminho** — o que acontece no checkpoint de shutdown. Em produção, o sintoma clássico desse cenário é o banco funcionar "normalmente" até o próximo restart.
 
### Resolução
 
Sincronizar todas as cópias **a partir da versão mais recente** — o ORA-00214 informa as duas versões, então a decisão é objetiva: propagar a 2994 (control02).
 
```bash
# 1. Garantir instância parada
SHUTDOWN ABORT;
 
# 2. Copiar o control02 (versão 2994) por cima das outras três cópias
!cp -f /u02/oradata/ORCL/control02.ctl /u02/oradata/ORCL/control01.ctl
!cp -f /u02/oradata/ORCL/control02.ctl /home/oracle/control_file.ctl
!cp -f /u02/oradata/ORCL/control02.ctl /u02/FRA/ORCL/control02.ctl
 
# 3. Subir normalmente
STARTUP;
SHOW PDBS;
-- Abre direto, sem recovery e sem RESETLOGS ✅
```
 
> 💡 Diferente do Incidente 2, aqui **não foi necessário recovery nem RESETLOGS** — porque o control02 era um control file **corrente** (o Oracle nunca parou de escrever nele), não uma cópia restaurada de backup.
 
### Causa raiz
 
Erro de digitação no caminho do `mv` (faltou a `/` inicial), que falhou silenciosamente para aquele arquivo específico. O control file "não movido" continuou vivo e atualizado; as cópias movidas e devolvidas ficaram defasadas.
 
### Aprendizado
 
- **O sentido da cópia importa:** no ORA-00214, sempre identificar a versão mais recente (o erro lista as duas) e propagar essa. Copiar a antiga por cima da nova joga fora atualizações e pode forçar um recovery desnecessário.
- Conferir o retorno de **cada** comando `mv`/`cp` em operações com múltiplos arquivos — um caminho sem `/` inicial falha só naquele arquivo e passa despercebido no meio da saída.
- No Linux, mover um arquivo aberto não interrompe a escrita do processo — o problema fica latente até o próximo reopen (checkpoint de shutdown/startup).
- A "versão" reportada pelo ORA-00214 é um contador interno de escrita do control file (binário) — não é legível via `strings`; a comparação de timestamp/tamanho no `ls -la` é suficiente para conferir a sincronização.
---
 
## Resumo dos incidentes
 
| # | Data | Erro | Causa raiz | Resolução |
|---|------|------|-----------|-----------|
| 1 | 02/07 | ORA-00205 no startup | control02.ctl na FRA (residual do lab de OMF) | `cp` para o caminho esperado |
| 2 | 04/07 | ORA-01152 no OPEN RESETLOGS | Control file de backup + redo da seq. 14 não aplicado, sem archived logs | RECOVER USING BACKUP CONTROLFILE aplicando o **online redo log** (redo03) + RESETLOGS |
| 3 | 04/07 | ORA-00214 no startup | `mv` com caminho errado → cópias com versões 2994 vs 2977 | `cp -f` da versão **mais recente** sobre as demais |
 
---
 
## Checklist pós-incidente (consolidado)
 
```sql
-- 1. Instância aberta?
SELECT instance_name, status FROM v$instance;
 
-- 2. Control files: configuração vs realidade
SELECT name FROM v$controlfile;
HOST ls -la /u02/oradata/ORCL/*.ctl /u02/FRA/ORCL/*.ctl /home/oracle/*.ctl
 
-- 3. Cópias sincronizadas? (timestamp e tamanho idênticos)
-- ls -la nas cópias — todas devem ter o mesmo horário de modificação
 
-- 4. PDBs abertos?
SHOW PDBS;
 
-- 5. Após qualquer OPEN RESETLOGS: backup imediato
ALTER DATABASE BACKUP CONTROLFILE TO '/home/oracle/control_file_ok.bkp' REUSE;
```
