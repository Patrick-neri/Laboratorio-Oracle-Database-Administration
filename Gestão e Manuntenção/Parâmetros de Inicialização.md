# LAB_REPORT — Parâmetros de Inicialização

**Data de execução:** 24/06/2026  
**Ambiente:** Oracle Database 19c — Oracle Linux  

---

## 1. Contexto e objetivo

Parâmetros de inicialização definem o comportamento da instância Oracle: quanta memória usar, quantas sessões simultâneas aceitar, onde estão os control files, qual o tamanho do bloco, etc. Esses parâmtros são de extrema importância, sem eles o banco não inicializa, pois não sabe o que deve ser feito.
E alterar um parâmetro errado ou no escopo errado pode derrubar a instância ou causar comportamento inesperado em produção.

Este lab cobre três áreas críticas:

- Como visualizar e alterar parâmetros estáticos e dinâmicos
- Como gerenciar PFILE e SPFILE — os arquivos que guardam esses parâmetros
- Como recuperar o SPFILE quando ele é perdido — cenário real de incidente

---

## 2. Sequência de execução

### 2.1 Parâmetros estáticos vs dinâmicos

Primeiro verifiquei se um parâmetro é estático ou dinâmico, para saber qual escopo utilizar:

```sql
-- Verificar se o parâmetro 'processes' é estático ou dinâmico
SELECT ISINSTANCE_MODIFIABLE FROM V$PARAMETER WHERE NAME = 'processes';

-- Verificar o valor atual
SHOW PARAMETER processes;
```

O resultado do `SELECT` na `V$PARAMETER` indica o tipo do parâmetro:

- `FALSE` = estático → precisa de `SCOPE=SPFILE` + reiniciar a instância para aplicar as alterações
- `TRUE`  = dinâmico → pode usar `SCOPE=BOTH` ou `SCOPE=MEMORY`

> <img width="659" height="228" alt="Resultado do SELECT em V$PARAMETER mostrando ISINSTANCE_MODIFIABLE para o parâmetro processes" src="https://github.com/user-attachments/assets/c0b8f99f-ed3a-4929-aa7e-0f65bcaa3e20" />

---

### 2.2 Alterando parâmetros — estático vs dinâmico

Após verificar o tipo de parâmetro, realizei testes de alteração comprovando o comportamento de cada escopo.

**Parâmetro estático — `processes`:**

```sql
-- Tentativa incorreta: alterar parâmetro estático na memória
ALTER SYSTEM SET processes = 500 SCOPE = MEMORY;
-- Resultado: ORA-02095 — parâmetro estático não pode ser alterado na memória

-- Forma correta: agendar para o próximo restart
ALTER SYSTEM SET processes = 500 SCOPE = SPFILE;

-- Confirmar que o valor em memória não mudou (ainda vale o anterior)
SHOW PARAMETER processes;
```

Como mostra a imagem, o parâmetro `processes` teve o valor alterado no SPFILE, mas não foi aplicado imediatamente — por ser estático, a mudança só vale após reiniciar a instância:

><img width="641" height="720" alt="image" src="https://github.com/user-attachments/assets/843376c3-1e31-4a0e-88a7-e3ef777820fe" />


**Parâmetro dinâmico — `db_cache_size`:**

```sql
-- Verificar se é dinâmico
SELECT ISINSTANCE_MODIFIABLE FROM V$PARAMETER WHERE NAME = 'db_cache_size';
-- Resultado: TRUE → pode alterar em memória imediatamente

-- Alterar na memória (efeito imediato, não persiste)
ALTER SYSTEM SET db_cache_size = 100M SCOPE = MEMORY;
SHOW PARAMETER db_cache_size;
-- Confirma o novo valor: 100M
```

> <img width="654" height="213" alt="Alteração do db_cache_size com SCOPE=MEMORY mostrando efeito imediato" src="https://github.com/user-attachments/assets/89577315-c947-4df9-bb45-08ace8ded04f" />

```sql
-- Desfazer a alteração em memória — volta ao valor do SPFILE
ALTER SYSTEM RESET db_cache_size SCOPE = MEMORY;
SHOW PARAMETER db_cache_size;
-- Volta ao valor original: 40M
```

> <img width="641" height="135" alt="ALTER SYSTEM RESET desfazendo a alteração em memória do db_cache_size" src="https://github.com/user-attachments/assets/43fa2b4d-c9d1-4d8b-aa85-e3fcea5b0cf2" />

---

### 2.3 Gerenciando PFILE e SPFILE

Agora fiz outros testes criando um PFILE a partir do SPFILE atual, uma medida de segurança antes de fazermos alterações importantes no SPFILE.
```sql
-- Criar PFILE a partir do SPFILE atual (backup de segurança)
CREATE PFILE = '/home/oracle/my_pfile.ora' FROM SPFILE;
```

Depois, iniciei a instância usando o PFILE criado.
```sql
-- Iniciar a instância usando o PFILE criado
SHUTDOWN IMMEDIATE;
STARTUP PFILE = '/home/oracle/my_pfile.ora';
```
Quando a instância é iniciada utilizando um PFILE, não existe associação com um SPFILE ativo. Por isso alterações com SCOPE=SPFILE ou SCOPE=BOTH não podem ser persistidas em um SPFILE inexistente.
><img width="646" height="630" alt="image" src="https://github.com/user-attachments/assets/4a8f7f09-7119-4f17-8be3-3c1c51f4737e" />

E quando alteramos parâmetros dinâmicos, a alteração é feita apenas na memória, ou seja, não fica persistente, como pode ver abaixo:
><img width="652" height="521" alt="image" src="https://github.com/user-attachments/assets/11f93c1a-2323-4e8a-873c-a5524034afd7" />


---

### 2.4 Cenários de recuperação do SPFILE (o mais importante do lab)

Em produção, o SPFILE pode ser perdido por erro humano, corrupção de disco ou manutenção mal executada. Este lab cobre três formas de recuperar.

---

**Cenário A — Recuperar SPFILE a partir da memória**

Nesse cenário simulei a perda do SPFILE, e como a instância estava ativa, uma maneira fácil de recuperar é por fazer uma cópia da memória.   
   - Criei o SPFILE em um caminho alternativo e depois copiei para o local padrão, pois o CREATE SPFILE permite especificar qualquer destino — o Oracle só usa automaticamente o arquivo que está em $ORACLE_HOME/dbs/spfileSID.ora.;
   
   
```sql
-- Simular perda do SPFILE (laboratório)
!rm -f /u01/app/oracle/product/19.0.0/dbhome_1/dbs/spfileorcl.ora
!rm -f /u01/app/oracle/product/19.0.0/dbhome_1/dbs/initorcl.ora

-- Criar SPFILE a partir do estado atual da memória
CREATE SPFILE = '/home/oracle/spfile_from_memory.ora' FROM MEMORY;

-- Copiar para o local padrão com o nome correto
!cp /home/oracle/spfile_from_memory.ora \
    /u01/app/oracle/product/19.0.0/dbhome_1/dbs/spfileorcl.ora

-- Confirmar que as alterações voltaram a funcionar
ALTER SYSTEM SET PROCESSES = 500 SCOPE = SPFILE;
```

- Na imagem abaixo, o resultado é que foi recuperado o SPFILE e então é possível fazer alterações novamente.

><img width="861" height="594" alt="image" src="https://github.com/user-attachments/assets/812f9ac7-0b8a-41d1-82c9-3755bd445325" />

---

**Cenário B — Recuperar SPFILE a partir do PFILE**

Nesse cenário simulei a perda do SPFILE com a instância inativa. Quando a instância é reiniciada ocorre uma falha.
  - O comando CREATE SPFILE requer apenas uma instância ativa. Portanto o estágio NOMOUNT é suficiente, pois nesse momento a SGA e os processos de background já foram criados; 
  - Usei a instruçção CREATE SPFILE= 'caminho onde salvei o pfile, que fiz anteriormente(no ponto 3.3)';
  - Com o SPFILE restaurado o banco é iniciado normalmente.
    
```sql
-- Simular perda e tentar reiniciar 
!rm -f /u01/app/oracle/product/19.0.0/dbhome_1/dbs/spfileorcl.ora
SHUTDOWN IMMEDIATE;
STARTUP;
-- 

-- Solução: subir no modo NOMOUNT pelo PFILE de backup
STARTUP NOMOUNT PFILE = '/home/oracle/my_pfile.ora';

-- Recriar o SPFILE a partir do PFILE
CREATE SPFILE FROM PFILE = '/home/oracle/my_pfile.ora';

-- Reiniciar normalmente
SHUTDOWN IMMEDIATE;
STARTUP;
```

 - Resultado:

><img width="803" height="565" alt="image" src="https://github.com/user-attachments/assets/9d583dae-6d32-4501-ad66-d834a3973836" />

---
**Cenário C — Recuperar SPFILE a partir do Alert Log **

Caso nenhuma do cenários anteriores seja possível, podemos recuperar os dados do SPFILE no Alert lLog.
   - Nessa situação acessei o Alert Log e identifiquei os parâmetros utilizados durante inicializações anteriores da instância.

```bash
# Localizar o Alert Log
# $ORACLE_BASE/diag/rdbms/orcl/orcl/trace/alert_orcl.log

# Buscar os parâmetros do último startup
grep -A 50 "System parameters" alert_orcl.log
```

> <img width="859" height="344" alt="Alert Log exibindo os parâmetros de inicialização registrados no último startup da instância" src="https://github.com/user-attachments/assets/4ffaa06e-c7c2-434b-97ec-e039d40e14d6" />

- Após isso, criei um arquivo PFILE e inseri os dados de inicialização nele;
```ini
# Criar PFILE manualmente com os valores extraídos do Alert Log
# /home/oracle/pfile_from_alert.ora
```
- Daí iniciei a instância pelo PFILE;
```sql
-- Iniciar pelo PFILE reconstruído
STARTUP PFILE = '/home/oracle/pfile_from_alert.ora';
```
- Depois, criei o SPFILE a partir do PFILE criado do Alert Log;
```sql
-- Recriar o SPFILE
CREATE SPFILE FROM PFILE = '/home/oracle/pfile_from_alert.ora';
-- ou: CREATE SPFILE FROM MEMORY;

-- Reiniciar normalmente pelo SPFILE
SHUTDOWN IMMEDIATE;
STARTUP;
```
  
  - Resultado: o banco pode ser iniciado normalmente.
><img width="813" height="650" alt="image" src="https://github.com/user-attachments/assets/d9b1f1a9-dd9f-4374-a9f6-c36b2ba3edf6" />

---

## 3. Erros encontrados e como resolvi

### Erro 1 — ORA-02095 ao tentar alterar `processes` com SCOPE=MEMORY

```
ORA-02095: specified initialization parameter cannot be modified
```

**Causa:** `processes` é um parâmetro estático — `ISINSTANCE_MODIFIABLE = FALSE` na V$PARAMETER.  
**Solução:** usar `SCOPE=SPFILE` e reiniciar a instância para valer.  
**Aprendizado:** sempre checar `ISINSTANCE_MODIFIABLE` antes de tentar alterar um parâmetro em memória.

---

### Erro 2 — STARTUP falhou após deletar SPFILE e PFILE

```
ORA-01078: failure in processing system parameters
LRM-00109: could not open parameter file '/u01/.../dbs/initorcl.ora'
```

**Causa:** Oracle procura SPFILE → PFILE na ordem padrão. Sem nenhum dos dois, o startup falha.  
**Solução:** Cenário B — subir com `STARTUP NOMOUNT PFILE=` e recriar o SPFILE.  
**Aprendizado:** nunca deletar SPFILE e PFILE ao mesmo tempo sem ter o PFILE em outro caminho.

---


