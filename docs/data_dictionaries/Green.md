# Análise de Dicionário de Dados: Green Taxi (LPEP) Trip Records

**Fonte:** TLC Data Dictionary - LPEP Trip Records 
**Data do Documento:** 18 de Março de 2025 

Este documento descreve a estrutura e os campos dos dados de viagem dos Táxis Verdes (Street Hail Livery - SHL). As informações são essenciais para a modelagem do Data Warehouse, o design do pipeline de ETL/ELT e a validação da qualidade dos dados.

---

## Tabela de Campos e Dicionário de Dados

A seguir, a descrição detalhada de cada campo presente no conjunto de dados.

| Nome do Campo | Descrição | Valores e Notas para Modelagem |
|---|---|---|
| `VendorID` | Um código que indica o provedor LPEP que forneceu o registro. | **Tipo:** Inteiro. <br> **Notas:** Campo categórico, ideal para uma tabela de dimensão `Dim_Vendor`. <br> **Valores:** `1`: Creative Mobile Technologies, LLC, `2`: Curb Mobility, LLC, `6`: Myle Technologies Inc. |
| `lpep_pickup_datetime` | A data e a hora em que o taxímetro foi acionado. | **Tipo:** Timestamp. <br> **Notas:** Campo chave para particionamento da tabela de fatos (ex: por ano/mês). |
| `lpep_dropoff_datetime` | A data e a hora em que o taxímetro foi desligado. | **Tipo:** Timestamp. |
| `store_and_fwd_flag` | Indica se o registro da viagem foi retido na memória do veículo antes de ser enviado, por falta de conexão com o servidor. | **Tipo:** String/Booleano. <br> **Valores:** `Y` = store and forward trip, `N` = not a store and forward trip. |
| `RatecodeID` | O código da tarifa final em vigor no final da viagem. | **Tipo:** Inteiro. <br> **Notas:** Chave estrangeira para a dimensão `Dim_RateCode`. O valor `99` deve ser tratado como Nulo/Desconhecido. <br> **Valores:** `1`: Standard rate, `2`: JFK, `3`: Newark, `4`: Nassau or Westchester, `5`: Negotiated fare, `6`: Group ride, `99`: Null/unknown. |
| `PULocationID` | TLC Taxi Zone em que o taxímetro foi acionado. | **Tipo:** Inteiro. <br> **Notas:** Chave estrangeira para a dimensão `Dim_Location`. |
| `DOLocationID` | TLC Taxi Zone em que o taxímetro foi desligado. | **Tipo:** Inteiro. <br> **Notas:** Chave estrangeira para a dimensão `Dim_Location`. |
| `passenger_count` | O número de passageiros no veículo. | **Tipo:** Inteiro. |
| `trip_distance` | A distância da viagem em milhas, conforme reportado pelo taxímetro. | **Tipo:** Numérico (Float/Decimal). |
| `fare_amount` | A tarifa calculada pelo taxímetro com base no tempo e na distância. | **Tipo:** Numérico (Float/Decimal). |
| `extra` | Extras e sobretaxas diversas. | **Tipo:** Numérico (Float/Decimal). |
| `mta_tax` | Imposto acionado automaticamente com base na tarifa em uso. | **Tipo:** Numérico (Float/Decimal). |
| `tip_amount` | Valor da gorjeta. Preenchido automaticamente para gorjetas de cartão de crédito; gorjetas em dinheiro não estão incluídas. | **Tipo:** Numérico (Float/Decimal). <br> **Regra de Negócio:** Análises de receita devem considerar que este campo não reflete gorjetas em dinheiro. |
| `tolls_amount` | Valor total de todos os pedágios pagos na viagem. | **Tipo:** Numérico (Float/Decimal). |
| `improvement_surcharge` | Sobretaxa de melhoria cobrada no início da corrida, implementada em 2015. | **Tipo:** Numérico (Float/Decimal). <br> **Notas:** Para dados anteriores a 2015, este campo não estará presente. |
| `total_amount` | O valor total cobrado dos passageiros, sem incluir gorjetas em dinheiro. | **Tipo:** Numérico (Float/Decimal). <br> **Regra de Negócio:** Este campo sub-representa o valor total transacionado em pagamentos com dinheiro. |
| `payment_type` | Um código numérico que significa como o passageiro pagou pela viagem. | **Tipo:** Inteiro. <br> **Notas:** Chave estrangeira para a dimensão `Dim_PaymentType`. <br> **Valores:** `0`: Flex Fare trip, `1`: Credit card, `2`: Cash, `3`: No charge, `4`: Dispute, `5`: Unknown, `6`: Voided trip. |
| `trip_type` | Um código que indica se a viagem foi uma chamada na rua (street-hail) ou por despacho (dispatch). | **Tipo:** Inteiro. <br> **Notas:** Campo fundamental que diferencia os Táxis Verdes. Ideal para a dimensão `Dim_TripType`. <br> **Valores:** `1`: Street-hail, `2`: Dispatch. |
| `congestion_surcharge` | Valor total coletado na viagem para a sobretaxa de congestionamento do estado de NY. | **Tipo:** Numérico (Float/Decimal). |
| `cbd_congestion_fee` | Taxa por viagem para a Zona de Alívio de Congestionamento da MTA, a partir de 5 de janeiro de 2025. | **Tipo:** Numérico (Float/Decimal). <br> **Notas:** Campo relevante para a evolução do esquema. Estará ausente em dados anteriores a esta data. |

---

## Considerações para Arquitetura de Dados

1.  **Modelo Dimensional (Star Schema):** Os dados são ideais para um modelo estrela. A tabela principal de viagens seria a **tabela de fatos**. As seguintes dimensões devem ser criadas:
    * `Dim_Vendor` (a partir de `VendorID`)
    * `Dim_RateCode` (a partir de `RatecodeID`)
    * `Dim_PaymentType` (a partir de `payment_type`)
    * `Dim_TripType` (a partir de `trip_type`, uma dimensão específica para este conjunto de dados)
    * `Dim_Location` (a partir de `PULocationID` e `DOLocationID`)
    * `Dim_Date` (a partir de `lpep_pickup_datetime` e `lpep_dropoff_datetime`)

2.  **Particionamento:** A tabela de fatos deve ser particionada por `lpep_pickup_datetime` (por exemplo, por ano/mês) para otimizar a performance de consultas e o gerenciamento do ciclo de vida dos dados.

3.  **Evolução do Esquema (Schema Evolution):** O modelo de dados deve ser projetado para acomodar a introdução de novos campos ao longo do tempo. A `improvement_surcharge` (2015)  e a `cbd_congestion_fee` (2025) são exemplos claros. Dados mais antigos terão valores nulos para estas colunas.

4.  **Regras de Negócio no ETL:** O pipeline de ETL deve implementar lógicas para:
    * Tratar o `RatecodeID = 99` como "Desconhecido".
    * Enriquecer os dados brutos com as informações das tabelas de dimensão.
    * Validar a consistência dos dados, especialmente os campos financeiros, aplicando as regras de negócio sobre gorjetas em dinheiro.

5.  **Qualidade de Dados e Análise:** É crucial que qualquer análise ou dashboard derivado destes dados inclua uma nota sobre a limitação dos campos `tip_amount` e `total_amount`, que não capturam gorjetas pagas em dinheiro. O campo `trip_type` é um diferenciador analítico chave em comparação com os dados dos táxis amarelos.