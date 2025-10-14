# Análise de Dicionário de Dados: Yellow Taxi Trip Records

**Fonte:** TLC Data Dictionary - Yellow Taxi Trip Records 
**Data do Documento:** 18 de Março de 2025 

Este documento descreve a estrutura, os campos e as regras de negócio dos dados de viagem dos táxis amarelos (Yellow Taxis). As informações aqui contidas são fundamentais para a modelagem do Data Warehouse, o design do pipeline de ETL/ELT e a garantia da qualidade dos dados.

---

## Tabela de Campos e Dicionário de Dados

A seguir, a descrição detalhada de cada campo presente no conjunto de dados.

| Nome do Campo | Descrição | Valores e Notas para Modelagem |
|---|---|---|
| `VendorID` | Um código que indica o provedor TPEP (provedor de tecnologia) que forneceu o registro. | **Tipo:** Inteiro. <br> **Notas:** Campo categórico, ideal para uma tabela de dimensão `Dim_Vendor`. <br> **Valores:** `1`: Creative Mobile Technologies, LLC, `2`: Curb Mobility, LLC, `6`: Myle Technologies Inc, `7`: Helix. |
| `tpep_pickup_datetime` | A data e a hora em que o taxímetro foi acionado (início da viagem). | **Tipo:** Timestamp. <br> **Notas:** Campo chave para particionamento da tabela de fatos (ex: por ano/mês). |
| `tpep_dropoff_datetime` | A data e a hora em que o taxímetro foi desligado (fim da viagem). | **Tipo:** Timestamp. |
| `passenger_count` | O número de passageiros no veículo. | **Tipo:** Inteiro. |
| `trip_distance` | A distância da viagem em milhas, conforme reportado pelo taxímetro. | **Tipo:** Numérico (Float/Decimal). |
| `RatecodeID` | O código da tarifa final em vigor no final da viagem. | **Tipo:** Inteiro. <br> **Notas:** Chave estrangeira para uma tabela `Dim_RateCode`. O valor `99` deve ser tratado como Nulo ou Desconhecido no ETL. <br> **Valores:** `1`: Standard rate, `2`: JFK, `3`: Newark, `4`: Nassau or Westchester, `5`: Negotiated fare, `6`: Group ride, `99`: Null/unknown. |
| `store_and_fwd_flag` | Indica se o registro da viagem foi retido na memória do veículo antes de ser enviado ao fornecedor por falta de conexão. | **Tipo:** String/Booleano. <br> **Valores:** `Y`: viagem "store and forward", `N`: não é uma viagem "store and forward". |
| `PULocationID` | TLC Taxi Zone em que o taxímetro foi acionado. | **Tipo:** Inteiro. <br> **Notas:** Chave estrangeira para a dimensão `Dim_Location` ou `Dim_TaxiZone`. |
| `DOLocationID` | TLC Taxi Zone em que o taxímetro foi desligado. | **Tipo:** Inteiro. <br> **Notas:** Chave estrangeira para a dimensão `Dim_Location` ou `Dim_TaxiZone`. |
| `payment_type` | Um código numérico que significa como o passageiro pagou pela viagem. | **Tipo:** Inteiro. <br> **Notas:** Chave estrangeira para a tabela `Dim_PaymentType`. <br> **Valores:** `0`: Flex Fare trip, `1`: Credit card, `2`: Cash, `3`: No charge, `4`: Dispute, `5`: Unknown, `6`: Voided trip. |
| `fare_amount` | A tarifa calculada pelo taxímetro com base no tempo e na distância. | **Tipo:** Numérico (Float/Decimal). |
| `extra` | Extras e sobretaxas diversas. | **Tipo:** Numérico (Float/Decimal). |
| `mta_tax` | Imposto que é acionado automaticamente com base na tarifa do taxímetro em uso. | **Tipo:** Numérico (Float/Decimal). |
| `tip_amount` | Valor da gorjeta. Este campo é preenchido automaticamente para gorjetas de cartão de crédito. Gorjetas em dinheiro não estão incluídas. | **Tipo:** Numérico (Float/Decimal). <br> **Regra de Negócio Crítica:** A ausência de gorjetas em dinheiro deve ser considerada em qualquer análise de receita ou de ganhos do motorista. |
| `tolls_amount` | Valor total de todos os pedágios pagos na viagem. | **Tipo:** Numérico (Float/Decimal). |
| `improvement_surcharge` | Sobretaxa de melhoria cobrada no início da corrida. Começou a ser cobrada em 2015. | **Tipo:** Numérico (Float/Decimal). <br> **Notas:** A análise histórica anterior a 2015 não conterá este campo. |
| `total_amount` | O valor total cobrado dos passageiros. Não inclui gorjetas em dinheiro. | **Tipo:** Numérico (Float/Decimal). <br> **Regra de Negócio Crítica:** Similar ao `tip_amount`, este campo sub-representa o valor total transacionado na viagem. |
| `congestion_surcharge` | Valor total coletado na viagem para a sobretaxa de congestionamento do estado de NY. | **Tipo:** Numérico (Float/Decimal). |
| `airport_fee` | Cobrado apenas para embarques nos aeroportos LaGuardia e John F. Kennedy. | **Tipo:** Numérico (Float/Decimal). <br> **Notas:** Aplicável apenas a um subconjunto de viagens. |
| `cbd_congestion_fee` | Taxa por viagem para a Zona de Alívio de Congestionamento da MTA, a partir de 5 de janeiro de 2025. | **Tipo:** Numérico (Float/Decimal). <br> **Notas:** Campo relevante para a evolução do esquema (schema evolution). Estará ausente ou nulo em dados anteriores a esta data. |

---

## Considerações para Arquitetura de Dados

1.  **Modelo Dimensional (Star Schema):** Os dados são altamente adequados para um modelo estrela. A tabela principal de viagens seria a **tabela de fatos**. As seguintes dimensões devem ser criadas:
    * `Dim_Vendor` (a partir de `VendorID`)
    * `Dim_RateCode` (a partir de `RatecodeID`)
    * `Dim_PaymentType` (a partir de `payment_type`)
    * `Dim_Location` (a partir de `PULocationID` e `DOLocationID`, usando a tabela de consulta de Zonas de Táxi da TLC)
    * `Dim_Date` (gerada a partir de `tpep_pickup_datetime` e `tpep_dropoff_datetime`)

2.  **Particionamento de Tabela:** A tabela de fatos de viagens deve ser particionada por `tpep_pickup_datetime` (por exemplo, por ano e mês) para otimizar a performance de consultas e a gerenciabilidade dos dados.

3.  **Evolução do Esquema (Schema Evolution):** O modelo deve ser flexível para acomodar a introdução de novos campos ao longo do tempo, como a `improvement_surcharge` em 2015 e a `cbd_congestion_fee` em 2025. Campos mais antigos terão valores nulos para estas colunas.

4.  **Regras de Negócio no ETL:** O pipeline de ETL deve implementar lógicas para:
    * Tratar o `RatecodeID = 99` como "Desconhecido".
    * Normalizar os valores `Y`/`N` do campo `store_and_fwd_flag` para booleano.
    * Enriquecer os dados com as informações das tabelas de dimensão.
    * Validar a consistência dos dados, especialmente os campos financeiros, tendo em mente as regras sobre gorjetas em dinheiro.

5.  **Qualidade de Dados:** É crucial destacar em qualquer dashboard ou relatório que os campos `tip_amount` e `total_amount` não incluem gorjetas pagas em dinheiro, o que representa uma limitação conhecida do conjunto de dados.