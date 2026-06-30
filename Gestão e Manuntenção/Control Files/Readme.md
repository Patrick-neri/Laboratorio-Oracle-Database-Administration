# Lab 03 — Gerenciando Control Files

**Área:** Gestão e Manutenção  

---

## Objetivo

Demonstrar o gerenciamento completo de control files no Oracle Database: multiplexação para alta disponibilidade, backup via cópia binária e via trace, e recuperação em dois cenários reais de perda — por cópia binária e por recriação com `CREATE CONTROLFILE`.

---

## O que são Control Files?

O control file é o arquivo binário mais crítico do Oracle Database. Ele registra a estrutura física do banco e é o primeiro arquivo lido durante o startup, na fase de MOUNT. Sem ele, o banco não monta.

O que o control file registra:
- Nome do banco de dados
- Localização de todos os datafiles e redo log files
- Timestamp de criação do banco
- Sequência do log atual
- Informações de checkpoint

**Detalhe:** o Oracle grava em todos os control files listados no parâmetro `CONTROL_FILES` simultaneamente, mas lê apenas o primeiro. Se qualquer um dos arquivos ficar indisponível durante a operação, a instância para imediatamente.

---

## O que este lab demonstra

| Habilidade | Relevância para o DBA |
|---|---|
| Verificar localização e quantidade de control files | Diagnóstico inicial de qualquer ambiente |
| Adicionar um terceiro control file (multiplexação) | Alta disponibilidade — boa prática obrigatória |
| Backup binário com `ALTER DATABASE BACKUP CONTROLFILE TO` | Recuperação rápida de control file perdido |
| Backup em trace com `ALTER DATABASE BACKUP CONTROLFILE TO TRACE` | Recuperação quando não há backup binário |
| Recuperação por cópia binária | Cenário mais comum em produção |
| Recuperação por `CREATE CONTROLFILE` | Pior cenário — perda total de todos os control files |
| Restaurar tempfiles após `CREATE CONTROLFILE` | Etapa obrigatória pós-recriação — frequentemente esquecida |

---

## Arquivos

```
03-control-files/
├── README.md                      ← este arquivo
├── LAB_REPORT.md                  ← documentação completa com prints

```

---

## Conceitos-chave

### Duas formas de backup do control file

| Método | Comando | Quando usar |
|---|---|---|
| Cópia binária | `ALTER DATABASE BACKUP CONTROLFILE TO '/caminho/arquivo.bkp'` | Recuperação rápida — copia e reinicia |
| Script SQL (trace) | `ALTER DATABASE BACKUP CONTROLFILE TO TRACE` | Quando não há backup binário — gera script de recriação |

### Sequência de recuperação por cópia binária

```
1. Banco com erro ORA-00210 (control file inacessível)
2. SHUTDOWN ABORT (único modo disponível com control file ausente)
3. Copiar backup binário para o local correto  →  !cp backup.bkp /u02/oradata/ORCL/control01.ctl
4. STARTUP (banco monta e abre normalmente)
```

### Sequência de recuperação por CREATE CONTROLFILE

```
1. Todos os control files perdidos → ORA-00205 no startup
2. Banco fica em estado NOMOUNT (instância ativa, sem montar)
3. Executar CREATE CONTROLFILE com todos os datafiles e redo logs
4. ALTER DATABASE OPEN RESETLOGS
5. ALTER PLUGGABLE DATABASE ALL OPEN
6. Restaurar tempfiles manualmente (não são registrados no CREATE CONTROLFILE)
```

### Por que RESETLOGS é obrigatório após CREATE CONTROLFILE?

O `CREATE CONTROLFILE` cria um novo arquivo sem histórico de SCN. O `RESETLOGS` reinicia a sequência de log, tornando o novo control file consistente com os datafiles existentes. Após `RESETLOGS`, faça backup imediato do banco.

---



> ⚠️ Partes deste script simulam perda de control files com `rm`. Execute **apenas em ambiente de laboratório**.

---

## Referências

- [Oracle Docs — Managing Control Files](https://docs.oracle.com/en/database/oracle/oracle-database/19/admin/managing-controlfiles.html)
- [Oracle Docs — CREATE CONTROLFILE](https://docs.oracle.com/en/database/oracle/oracle-database/19/sqlrf/CREATE-CONTROLFILE.html)
- [Oracle Docs — V$CONTROLFILE](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/V-CONTROLFILE.html)
- Mentoria DBAOCM 
