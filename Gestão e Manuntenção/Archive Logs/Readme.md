# Lab — Gerenciando Archived Redo Logs

**Área:** Gestão e Manutenção  


---

## Objetivo

Demonstrar a ativação do modo ARCHIVELOG, a configuração de múltiplos destinos de arquivamento (multiplexação de archived logs), e a personalização do formato de nomenclatura dos arquivos via `LOG_ARCHIVE_FORMAT`.

---

## O que é um Archived Redo Log?

Um archived redo log file é uma cópia de um dos membros de um grupo de redo, contendo os mesmos registros de redo e o mesmo log sequence number do grupo de origem. O processo de salvar esses dados é chamado de **arquivamento**, executado pelo processo de background **ARCn**.

Assim que ocorre um log switch, o grupo de redo se torna elegível para arquivamento e permanece em status `ACTIVE` até que o ARCn copie seus registros para um archived redo log.

---

## NOARCHIVELOG vs ARCHIVELOG

| | NOARCHIVELOG | ARCHIVELOG |
|---|---|---|
| Arquivamento do redo | Não ocorre | Ocorre a cada log switch |
| Protege contra | Falha de instância | Falha de instância **e** de mídia |
| Recovery possível | Apenas até o último backup (banco fechado) | Até o momento exato da falha (point-in-time) |
| Backup com banco aberto | Não | Sim |
| Grupo de redo cheio | Fica disponível para sobrescrita assim que INACTIVE | Fica ACTIVE até ser arquivado — não pode ser sobrescrito antes |

> ⚠️ Após mudar de NOARCHIVELOG para ARCHIVELOG, um backup full é obrigatório — backups anteriores feitos em NOARCHIVELOG deixam de ser válidos para recovery.

---

## O que este lab demonstra

| Habilidade | Relevância para o DBA |
|---|---|
| Habilitar o modo ARCHIVELOG | Pré-requisito para qualquer estratégia de backup online e recovery point-in-time |
| Verificar modo com `ARCHIVE LOG LIST` e `V$DATABASE` | Diagnóstico padrão em qualquer ambiente |
| Configurar múltiplos destinos (`LOG_ARCHIVE_DEST_n`) | Multiplexação de archived logs — redundância contra perda de disco |
| Usar `USE_DB_RECOVERY_FILE_DEST` | Integração do arquivamento com a FRA |
| Personalizar `LOG_ARCHIVE_FORMAT` | Padronização de nomenclatura em ambientes com múltiplos destinos/instâncias |
| Identificar parâmetro estático via `ISINSTANCE_MODIFIABLE` | Mesma disciplina aplicada em todo parâmetro antes de alterar |

---

## Arquivos

```
06-archived-redo-logs/
├── README.md       ← este arquivo
├── LAB_REPORT.md   ← documentação completa com prints e erros encontrados
└── scripts/
    └── archived_redo_logs_lab.sql   ← script completo do lab
```

---

## Conceito-chave: variáveis de nomenclatura do LOG_ARCHIVE_FORMAT

| Variável | Significado |
|---|---|
| `%s` | Número da sequência do log (minúsculo — sem padding) |
| `%S` | Número da sequência do log (maiúsculo — tamanho fixo, zeros à esquerda) |
| `%t` | Número da thread |
| `%T` | Número da thread (maiúsculo — tamanho fixo) |
| `%a` | Id de ativação |
| `%d` | Id do banco de dados |
| `%r` | Id do resetlogs |

> ⚠️ O nome deve conter pelo menos `%s` (ou `%S`), `%t` (ou `%T`) e `%r`. Arquivos armazenados na FRA não seguem esse padrão de nomenclatura.

---

## Como reproduzir

```bash
sqlplus / as sysdba
@scripts/archived_redo_logs_lab.sql
```

> ⚠️ A alteração do modo de arquivamento exige `SHUTDOWN IMMEDIATE` — execute apenas em ambiente de laboratório.

---

## Referências

- [Oracle Docs — Managing Archived Redo Log Files](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-archived-redo-log-files.html)
- [Oracle Docs — LOG_ARCHIVE_FORMAT](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/LOG_ARCHIVE_FORMAT.html)
- Mentoria DBAOCM 
