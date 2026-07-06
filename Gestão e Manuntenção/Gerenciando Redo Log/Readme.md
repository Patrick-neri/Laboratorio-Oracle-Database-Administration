# Lab — Gerenciando Redo Logs

**Área:** Gestão e Manutenção  

---

## Objetivo

Demonstrar o gerenciamento completo dos Online Redo Logs no Oracle Database: consulta de grupos e membros, criação com e sem OMF, adição de membros, renomeação e movimentação de arquivos, remoção de grupos e membros, e controle de geração de redo por nível (CDB, PDB e tablespace).

---

## O que são Redo Logs?

Os Online Redo Logs são a estrutura mais crucial para recuperação do Oracle. Registram todas as alterações feitas no banco assim que ocorrem. O processo LGWR (Log Writer) escreve os dados do Redo Log Buffer para os arquivos de redo de forma circular — quando um grupo enche, passa para o próximo.

---

## Os quatro status de um grupo de redo

| Status | Descrição |
|---|---|
| `CURRENT` | Grupo sendo gravado pelo LGWR no momento |
| `ACTIVE` | Necessário para instance recovery — não pode ser sobrescrito nem dropado |
| `INACTIVE` | Não necessário para recovery — pode ser sobrescrito e dropado |
| `UNUSED` | Grupo recém-criado, nunca utilizado pelo LGWR |

> ⚠️ Para dropar um grupo `ACTIVE`, é necessário forçar um log switch e depois um checkpoint para que ele passe a `INACTIVE`.

---

## O que este lab demonstra

| Habilidade | Relevância para o DBA |
|---|---|
| Consultar grupos e membros (`V$LOG`, `V$LOGFILE`) | Diagnóstico inicial de qualquer problema de redo |
| Forçar log switch e checkpoint | Necessário antes de operações de manutenção |
| Criar grupo sem OMF (caminhos explícitos) | Controle total sobre localização dos arquivos |
| Criar grupo com OMF (`db_create_online_log_dest_n`) | Multiplexação automática em dois destinos |
| Adicionar membro a grupo existente | Aumentar redundância sem criar novo grupo |
| Renomear e mover membros (`RENAME FILE ... TO`) | Reorganização de disco — exige banco em MOUNT |
| Remover grupo (`DROP LOGFILE GROUP`) | Limpeza de grupos desnecessários |
| Remover membro (`DROP LOGFILE MEMBER`) | Reduzir membros mantendo o grupo |
| Configurar FORCE LOGGING / NOLOGGING | Controle de geração de redo por nível |

---

## Conceitos-chave

### Multiplexação de redo logs

Cada grupo pode ter múltiplos membros (arquivos físicos) em discos diferentes. O LGWR grava em todos os membros do grupo ativo simultaneamente. Se um membro falhar, o Oracle continua com os demais — mas registra aviso no Alert Log.

```
Grupo 1: membro_disco_A + membro_disco_B  ← LGWR grava nos dois simultaneamente
Grupo 2: membro_disco_A + membro_disco_B
Grupo 3: membro_disco_A + membro_disco_B
```

### Sequência para mover/renomear membros

```
1. SHUTDOWN IMMEDIATE
2. Mover o arquivo no SO  (!mv origem destino)
3. STARTUP MOUNT
4. ALTER DATABASE RENAME FILE 'origem' TO 'destino'
5. ALTER DATABASE OPEN
```

### Regras para DROP de grupo

```
Grupo pode ser dropado?
 ├─ Status CURRENT → ❌ Forçar SWITCH LOGFILE primeiro
 ├─ Status ACTIVE  → ❌ SWITCH LOGFILE + CHECKPOINT para INACTIVE
 └─ Status INACTIVE → ✅ Pode dropar
```


---

## Arquivos

```
05-redo-logs/
├── README.md       ← este arquivo
├── LAB_REPORT.md   ← documentação completa com prints e erros encontrados

```



---

## Referências

- [Oracle Docs — Managing the Redo Log](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-the-redo-log.html)
- [Oracle Docs — V$LOG](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-LOG.html)
- [Oracle Docs — V$LOGFILE](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-LOGFILE.html)
- Mentoria DBAOCM
