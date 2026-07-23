# Oracle Data Guard — Standby Físico (19c)
**Tipo:** Laboratório pessoal · portfólio  
**Área:** Alta Disponibilidade e DR (HA/DR)  


---

## Objetivo
Construir do zero um ambiente **Oracle Data Guard** com um banco **primário** (`MERC`, em `nerv01`) e um **standby físico** (`MERC02`, em `nerv02`), demonstrando a criação do standby via **RMAN `DUPLICATE ... FROM ACTIVE DATABASE`**, a configuração do **transporte de redo**, o início do **Managed Recovery (MRP)** e a validação da **aplicação de redo**.

---

## O que é Data Guard?
O **Oracle Data Guard** mantém uma ou mais cópias **standby** de um banco primário, sincronizadas pelo envio e aplicação de **redo**. É a solução de **alta disponibilidade e disaster recovery** do Oracle: se o primário falha, um standby assume (failover); em manutenções planejadas, os papéis são trocados (switchover). Diferente de um backup — que é uma foto do passado a ser restaurada — o standby é um banco **vivo**, continuamente atualizado.

Equivalente no SQL Server: **Always On Availability Groups** / Log Shipping.

---

## Topologia

| Papel | Host | IP | DB_NAME | DB_UNIQUE_NAME | Serviço TNS |
|---|---|---|---|---|---|
| **Primário** | `nerv01` | 192.168.0.121 | `MERC` | `MERC` | `MERC` |
| **Standby** | `nerv02` | 192.168.0.122 | `MERC` | `MERC02` | `MERC02` |

Ambiente: 2× VM **Oracle Linux 7**, Oracle Database **19c EE (19.3.0.0.0)**, OMF habilitado.

---

## O que este lab demonstra

| Habilidade | Relevância para o DBA |
|---|---|
| Preparar o primário (ARCHIVELOG, FORCE LOGGING) | Pré-requisitos obrigatórios de qualquer Data Guard |
| Ajustar listener/TNS para o IP real e `tnsping` cruzado | Base do transporte de redo entre os nós |
| Copiar password file e spfile (`scp`) | Autenticação do redo transport / RMAN AUXILIARY |
| Preparar diretórios e `NOMOUNT` do standby | Destino do `DUPLICATE` |
| `RMAN DUPLICATE ... FOR STANDBY FROM ACTIVE DATABASE` | Criar o standby sem backup prévio, direto da instância ativa |
| Configurar `LOG_ARCHIVE_CONFIG` / `LOG_ARCHIVE_DEST_2` | Transporte de redo do primário para o standby |
| `FAL_SERVER`, `STANDBY_FILE_MANAGEMENT`, MRP | Aplicação de redo e resolução de gaps no standby |
| Validar sincronização (`V$DATAGUARD_PROCESS`, `V$ARCHIVED_LOG`) | Provar que o DG está operando ponta a ponta |

---

## Pré-requisitos (ver seção *Gestão e Manutenção*)
Data Guard depende de conceitos já documentados em outra área do portfólio:
- **ARCHIVELOG** e **Archived Redo Logs** — o standby se alimenta do redo arquivado.
- **Redo Log** e **FORCE LOGGING** — garantem que toda alteração gere redo replicável.
- **OMF** — simplifica os caminhos de datafile no duplicate.
- **RMAN** — motor do `DUPLICATE ... FROM ACTIVE DATABASE`.

---

## Resultado
Standby `MERC02` no papel **PHYSICAL STANDBY**, modo **MAXIMUM PERFORMANCE** (ASYNC), com o **MRP0** aplicando redo e alcançando o primário (`Media Recovery Waiting for T-1.S-30 (in transit)`). Todo o passo a passo, os erros reais encontrados (com destaque para o **ORA-16047 / FAL DGID mismatch**) e as evidências dos alert logs estão no relatório.

---

## Arquivos
```
Data Guard/
├── README.md                    ← este arquivo
├── LAB_REPORT - Data Guard.md   ← passo a passo, erros, evidências e prints

```

---

## Próximos passos
- ⬜ Criar **Standby Redo Logs (SRL)** nos dois lados e habilitar **Real-Time Apply**.
- ⬜ Testar **switchover** e **failover**.
- ⬜ Configurar o **Data Guard Broker** (`dgmgrl`).

---

## Referências
- [Oracle Docs — Data Guard Concepts and Administration (19c)](https://docs.oracle.com/en/database/oracle/oracle-database/19/sbydb/index.html)
- [Oracle Docs — RMAN DUPLICATE for standby](https://docs.oracle.com/en/database/oracle/oracle-database/19/bradv/duplicating-databases.html)
