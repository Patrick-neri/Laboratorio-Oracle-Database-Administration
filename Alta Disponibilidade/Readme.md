# Alta Disponibilidade e Disaster Recovery (HA/DR)

Esta área reúne os labs voltados à **continuidade de negócio** do Oracle Database: manter o banco disponível diante de falhas e permitir recuperação em outro servidor. Diferente da manutenção do dia a dia (ver *Gestão e Manutenção*), aqui o foco é **redundância, replicação e tolerância a desastres**.

## Conteúdo

| Tópico | Descrição | Status |
|---|---|---|
| **Data Guard** | Standby físico 19c: RMAN active duplicate, transporte e aplicação de redo, MRP | 🟢 Núcleo concluído |
| RMAN Backup e Restore | Backup, restore e recuperação com RMAN | ⬜ planejado |

## Conceitos-chave da área

- **Standby ≠ backup:** um backup é uma foto do passado a ser restaurada; um **standby** é um banco vivo, continuamente atualizado por redo, pronto para assumir.
- **Switchover vs Failover:** switchover é a troca **planejada** de papéis (manutenção); failover é a assunção **não planejada** após falha do primário.
- **Modos de proteção:** Maximum Performance (ASYNC), Maximum Availability e Maximum Protection (SYNC) — trade-off entre desempenho e zero perda de dados.

## Pré-requisitos (em *Gestão e Manutenção*)
Estes labs se apoiam em fundamentos já documentados: **ARCHIVELOG**, **Archived Redo Logs**, **Redo Log / Force Logging**, **OMF** e **RMAN**.

## Equivalência SQL Server
Always On Availability Groups · Log Shipping · Database Mirroring (legado).
