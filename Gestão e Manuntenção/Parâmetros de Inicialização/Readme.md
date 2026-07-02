# Laboratório — Parâmetros de Inicialização

**Área:** Gestão e Manutenção  

---

## Objetivo

Demonstrar como visualizar, alterar e recuperar parâmetros de inicialização do Oracle Database, cobrindo a diferença entre parâmetros estáticos e dinâmicos, o gerenciamento de PFILE e SPFILE, e os três cenários de recuperação do SPFILE em ambiente de produção.

---

## O que este lab demonstra

| Habilidade | Relevância para o DBA |
|---|---|
| Identificar parâmetros estáticos vs dinâmicos | Evita erros de escopo que derrubam instâncias |
| Usar `SCOPE=MEMORY / SPFILE / BOTH` corretamente | Padrão em qualquer alteração de parâmetro em produção |
| Criar PFILE a partir do SPFILE | Backup obrigatório antes de manutenções críticas |
| Recuperar SPFILE da memória | Resolve incidente sem restart — impacto zero |
| Recuperar SPFILE pelo PFILE | Resolve startup failure em minutos |
| Recuperar SPFILE pelo Alert Log | Último recurso — cobre o pior cenário possível |

---

## Conceitos-chave

### Os três escopos do ALTER SYSTEM

| SCOPE | Efeito imediato | Persiste após restart | Quando usar |
|---|---|---|---|
| `MEMORY` | ✅ Sim | ❌ Não | Teste temporário |
| `SPFILE` | ❌ Não | ✅ Sim | Parâmetros estáticos |
| `BOTH` | ✅ Sim | ✅ Sim | Parâmetros dinâmicos |

### PFILE vs SPFILE

| Característica | PFILE (`initSID.ora`) | SPFILE (`spfileSID.ora`) |
|---|---|---|
| Formato | Texto — editável manualmente | Binário — alterado via `ALTER SYSTEM` |
| Persiste alterações? | Não — edição manual necessária | Sim — `SCOPE=SPFILE` ou `SCOPE=BOTH` |
| Usado em produção? | Raramente — recuperação e testes | Sim — padrão em ambientes modernos |

### Árvore de decisão — recuperação do SPFILE

```
Banco está UP?
 ├─ Sim  →  CREATE SPFILE FROM MEMORY                          (Cenário A)
 └─ Não  →  Tenho PFILE salvo?
             ├─ Sim  →  STARTUP NOMOUNT PFILE + CREATE SPFILE  (Cenário B)
             └─ Não  →  Alert Log → PFILE manual → SPFILE      (Cenário C)
```

---

## Arquivos

```
04-param-inicializacao/
├── README.md       ← este arquivo
├── LAB_REPORT.md   ← documentação completa com prints e erros encontrados

```

---

## Como reproduzir

**Pré-requisito:** Oracle Database 19c com acesso como `SYSDBA`.

```bash
. oraenv
SID
sqlplus / as sysdba
*Seguir passo a passo do lab_report*
```

> ⚠️ Os cenários de recuperação simulam perda de arquivos com `rm`. Execute **apenas em ambiente de laboratório**.

---

## Referências

- [Oracle Docs — Initialization Parameters](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/initialization-parameters.html)
- [Oracle Docs — ALTER SYSTEM](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/ALTER-SYSTEM.html)
- [Oracle Docs — V$PARAMETER](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-PARAMETER.html)
- Mentoria DBAOCM 
