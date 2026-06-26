# Lab — Oracle Managed Files (OMF)


## Objetivo

Demonstrar como o OMF simplifica a administração de arquivos do banco de dados Oracle, eliminando a necessidade de especificar caminhos manualmente na criação de tablespaces, redo logs e arquivos de backup. Cobre os três parâmetros de destino do OMF e a configuração da Fast Recovery Area (FRA).

---

## O que este lab demonstra

| Habilidade | Relevância para o DBA |
|---|---|
| Criar tablespace com e sem OMF | Entender o impacto do `db_create_file_dest` no dia a dia |
| Adicionar datafiles via OMF | Simplificação de rotinas de expansão de espaço |
| Multiplexar redo logs via OMF | Alta disponibilidade sem gestão manual de caminhos |
| Configurar `DB_CREATE_ONLINE_LOG_DEST_n` | Redundância de redo logs em discos separados |
| Configurar a Fast Recovery Area (FRA) | Centralizar backups, archived logs e flashback logs |
| Verificar uso da FRA com `V$RECOVERY_AREA_USAGE` | Monitoramento preventivo de espaço de backup |

---

## Parâmetros OMF cobertos

| Parâmetro | Controla | Comportamento sem OMF |
|---|---|---|
| `db_create_file_dest` | Datafiles e tempfiles de tablespaces | Obrigatório especificar caminho no `CREATE TABLESPACE` |
| `db_create_online_log_dest_n` | Redo log members (multiplexação) | Obrigatório especificar caminho no `ALTER DATABASE ADD LOGFILE` |
| `db_recovery_file_dest` | FRA: backups, archived logs, flashback | Obrigatório especificar caminho em cada operação de backup |

---

## Arquivos

```
02-oracle-managed-files/
├── README.md                    ← este arquivo
├── LAB_REPORT.md                ← documentação completa com prints
└── scripts/
    └── omf_lab.sql              ← script completo do lab com comentários
```

---

## Como reproduzir

**Pré-requisitos:**
- Oracle Database 19c com instância ativa
- Acesso como `SYSDBA`
- Diretórios criados no SO antes de configurar os parâmetros:

```bash
mkdir /home/oracle/oradata
mkdir /u02/FRA
```

> ⚠️ O Oracle valida a existência do diretório no momento da configuração do parâmetro. Configurar um caminho inexistente gera `ORA-01262`.

**Executar o script:**

```bash
sqlplus / as sysdba
```

---

## Conceito-chave: hierarquia de destino dos arquivos OMF

Quando o Oracle cria um arquivo gerenciado, ele segue esta ordem de prioridade para redo logs:

```
1. DB_CREATE_ONLINE_LOG_DEST_1 até _5  (se definido)
2. DB_CREATE_FILE_DEST                 (fallback)
3. Erro ORA-02236                      (nenhum definido)
```

Para datafiles e tempfiles:

```
1. DB_CREATE_FILE_DEST   (se definido)
2. Erro ORA-02199        (nenhum definido)
```

---

## Comportamento do OMF nos arquivos criados

Arquivos gerenciados pelo Oracle têm nomenclatura automática no formato:

```
/destino/DB_UNIQUE_NAME/datafile/o1_mf_<tsname>_<tag>_.dbf
/destino/DB_UNIQUE_NAME/onlinelog/o1_mf_<group>_<tag>_.log
```

Características automáticas dos arquivos OMF:
- **AUTOEXTENSIBLE = YES** por padrão
- **MAXSIZE = 32767.9844 MB** (máximo para o sistema de arquivos)
- Remoção automática quando o objeto é dropado — sem arquivos órfãos

---

## Referências

- [Oracle Docs — Oracle Managed Files](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/using-oracle-managed-files.html)
- [Oracle Docs — DB_CREATE_FILE_DEST](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DB_CREATE_FILE_DEST.html)
- [Oracle Docs — Fast Recovery Area](https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/configuring-rman-client-basic.html)
- Mentoria DBAOCM — Aula V9
