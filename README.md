# ETL & Data Modeling — Home Credit Default Risk

## 1. Objetivo do ETL

Esta etapa tem como objetivo estruturar os dados brutos do dataset Home Credit Default Risk em um banco relacional Oracle, preservando a granularidade original das tabelas e preparando a base para análises exploratórias, engenharia de atributos e modelagem preditiva.

O processo foi desenhado para simular um cenário corporativo real, onde diferentes fontes históricas de crédito precisam ser integradas antes da construção de um modelo de classificação binária para previsão de inadimplência.

---

## 2. Contexto dos Dados

O dataset possui uma estrutura relacional implícita, composta por uma tabela principal de solicitações de crédito e tabelas auxiliares com históricos financeiros dos clientes.

A tabela central é `HC_APPLICATION`, que representa a solicitação atual de crédito. A partir dela, existem dois grandes blocos históricos:

- Histórico externo de crédito, vindo de outras instituições financeiras;
- Histórico interno de aplicações anteriores na Home Credit.

---

## 3. Granularidade das Tabelas

| Tabela | Grão | Chave Principal | Relacionamento |
|---|---|---|---|
| `HC_APPLICATION` | 1 linha por cliente/solicitação atual | `SK_ID_CURR` | Tabela central |
| `HC_BUREAU` | 1 linha por crédito anterior em outra instituição | `SK_ID_BUREAU` | `SK_ID_CURR` |
| `HC_BUREAU_BALANCE` | 1 linha por mês de saldo do crédito externo | `SK_ID_BUREAU`, `MONTHS_BALANCE` | `SK_ID_BUREAU` |
| `HC_PREVIOUS_APPLICATION` | 1 linha por solicitação anterior na Home Credit | `SK_ID_PREV` | `SK_ID_CURR` |
| `HC_POS_CASH_BALANCE` | 1 linha por mês de saldo POS/CASH | `SK_ID_PREV`, `MONTHS_BALANCE` | `SK_ID_PREV` |
| `HC_INSTALLMENTS_PAYMENTS` | 1 linha por parcela paga/devida | `SK_ID_PREV`, `NUM_INSTALMENT_NUMBER` | `SK_ID_PREV` |
| `HC_CREDIT_CARD_BALANCE` | 1 linha por mês de saldo de cartão | `SK_ID_PREV`, `MONTHS_BALANCE` | `SK_ID_PREV` |

---

## 4. Decisões de Modelagem

### 4.1 Unificação de `application_train` e `application_test`

As bases `application_train.csv` e `application_test.csv` possuem estrutura semelhante, porém apenas a base de treino contém a variável alvo `TARGET`.

Para simplificar a estrutura relacional, ambas foram unificadas na tabela `HC_APPLICATION`, com a criação da coluna `DATASET_TYPE`.

- `TRAIN`: registros com variável `TARGET`;
- `TEST`: registros sem variável `TARGET`.

Essa decisão facilita consultas, auditoria da base e futuras etapas de feature engineering.

---

### 4.2 Preservação da granularidade original

As tabelas foram carregadas mantendo sua granularidade original, evitando agregações prematuras.

Essa abordagem permite:

- rastreabilidade dos dados;
- flexibilidade na criação de features;
- validação dos relacionamentos;
- construção posterior de variáveis agregadas por cliente.

---

### 4.3 Camada Raw Relacional

A primeira camada do ETL foi implementada como uma camada raw estruturada em Oracle.

O objetivo dessa camada não é ainda entregar a tabela final de modelagem, mas sim organizar as fontes originais em um formato relacional, com chaves, tipos de dados e relacionamentos documentados.

---

### 4.4 Tratamento de constraints

Foram aplicadas chaves primárias nas principais tabelas para garantir unicidade dos registros.

As chaves estrangeiras foram utilizadas de forma seletiva. Durante a carga, algumas tabelas auxiliares apresentaram registros históricos que não encontravam correspondência direta na amostra principal carregada.

Por esse motivo, na camada raw, a integridade referencial foi tratada prioritariamente nas consultas e views analíticas, utilizando joins controlados.

---

## 5. Estratégia de Carga

A ingestão foi realizada com Python, utilizando:

- `pandas` para leitura dos arquivos CSV;
- `oracledb` para conexão e escrita no Oracle;
- leitura em chunks para tabelas volumosas;
- commits por lote;
- tratamento de valores nulos e infinitos.

As tabelas menores foram carregadas diretamente em memória. As tabelas maiores foram processadas em lotes para evitar estouro de memória.

---

## 6. Tratamentos Aplicados

Durante a ingestão foram aplicados os seguintes tratamentos:

- substituição de valores `NaN` por `NULL`;
- substituição de valores infinitos por `NULL`;
- conversão de tipos para compatibilidade com Oracle;
- criação da coluna `DATASET_TYPE`;
- padronização dos nomes das tabelas;
- seleção inicial de colunas relevantes para o case.

---

## 7. Principais Desafios Técnicos

### 7.1 Volume de Dados

Algumas tabelas, como `HC_BUREAU_BALANCE`, possuem milhões de registros. A leitura integral desses arquivos em memória causava erro de memória no pandas.

Solução adotada:

- leitura com `chunksize`;
- carga incremental por lote;
- commits por chunk.

---

### 7.2 Conversão de valores nulos

Durante a carga inicial, valores `NaN` em colunas numéricas causaram erro de conversão no Oracle.

Solução adotada:

- substituição explícita de `NaN`, `pd.NA`, `inf` e `-inf` por `None`.

---

### 7.3 Integridade referencial

Durante a carga de tabelas auxiliares, algumas constraints de chave estrangeira bloquearam registros que não possuíam correspondência direta na tabela mãe.

Solução adotada:

- manter a carga raw mais flexível;
- tratar integridade e relacionamento na camada analítica;
- documentar essa decisão como uma escolha de arquitetura.

---

## 8. Ordem de Carga

A ordem de carga respeitou a dependência lógica entre as tabelas:

1. `HC_APPLICATION`
2. `HC_BUREAU`
3. `HC_BUREAU_BALANCE`
4. `HC_PREVIOUS_APPLICATION`
5. `HC_POS_CASH_BALANCE`
6. `HC_INSTALLMENTS_PAYMENTS`
7. `HC_CREDIT_CARD_BALANCE`

---

## 9. Validações Pós-Carga

Após a ingestão, foram executadas consultas de validação para verificar:

- volume carregado por tabela;
- existência de chaves duplicadas;
- percentual de registros nulos em colunas críticas;
- distribuição da variável alvo;
- taxa de inadimplência por segmento;
- consistência entre tabelas relacionadas.

Exemplo:

```sql
SELECT COUNT(*) FROM HC_APPLICATION;
SELECT COUNT(*) FROM HC_BUREAU;
SELECT COUNT(*) FROM HC_BUREAU_BALANCE;
SELECT COUNT(*) FROM HC_PREVIOUS_APPLICATION;
SELECT COUNT(*) FROM HC_POS_CASH_BALANCE;
SELECT COUNT(*) FROM HC_INSTALLMENTS_PAYMENTS;
SELECT COUNT(*) FROM HC_CREDIT_CARD_BALANCE;