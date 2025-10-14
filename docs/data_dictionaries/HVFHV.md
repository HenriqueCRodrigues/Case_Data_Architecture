# Análise de Dicionário de Dados para Arquitetura: High Volume FHV (HVFHS) Trip Records

**Fonte:** Data Dictionary - High Volume FHV Trip Records
**Data do Documento:** 18 de Março de 2025

Este documento descreve a estrutura dos dados de viagem para "High-Volume For-Hire Services" (HVFHS). Esta é uma categoria de licença criada em 1º de fevereiro de 2019, para empresas que despacham mais de 10.000 viagens por dia em Nova York sob uma única marca. Cada linha neste conjunto de dados representa uma única viagem despachada por uma dessas bases.

---

## Tabela de Campos e Dicionário de Dados

A seguir, a descrição detalhada de cada campo presente no conjunto de dados.

| Nome do Campo | Descrição | Valores e Notas para Modelagem |
|---|---|---|
| `hvfhs_license_num` | O número da licença da TLC da empresa ou base HVFHS. | **Tipo:** String. <br> **Notas:** Chave primária para uma dimensão `Dim_Company`. <br> **Valores (em Set/2019):** `HV0002`: Juno, `HV0003`: Uber, `HV0004`: Via, `HV0005`: Lyft. |
| `dispatching_base_num` | O número da licença da base da TLC que despachou a viagem. | **Tipo:** String. <br> **Notas:** Chave estrangeira para a dimensão `Dim_Base`. |
| `originating_base_num` | O número da base que recebeu a solicitação original da viagem. | **Tipo:** String. <br> **Notas:** Pode ser o mesmo que `dispatching_base_num`. |
| `request_datetime` | Data e hora em que o passageiro solicitou a viagem. | **Tipo:** Timestamp. |
| `on_scene_datetime` | Data e hora em que o motorista chegou ao local de embarque. | **Tipo:** Timestamp. <br> **Notas:** Aplicável apenas para Veículos . |
| `pickup_datetime` | A data e a hora do embarque da viagem. | **Tipo:** Timestamp. <br> **Notas:** Campo ideal para particionamento da tabela de fatos. |
| `dropoff_datetime` | A data e a hora do desembarque da viagem. | **Tipo:** Timestamp. |
| `PULocationID` | A Zona de Táxi da TLC onde a viagem começou. | **Tipo:** Inteiro. <br> **Notas:** Chave estrangeira para a dimensão `Dim_Location`. |
| `DOLocationID` | A Zona de Táxi da TLC onde a viagem terminou. | **Tipo:** Inteiro. <br> **Notas:** Chave estrangeira para a dimensão `Dim_Location`. |
| `trip_miles` | Total de milhas da viagem do passageiro. | **Tipo:** Numérico (Float/Decimal). |
| `trip_time` | Tempo total da viagem do passageiro em segundos. | **Tipo:** Inteiro. |
| `base_passenger_fare` | Tarifa base do passageiro antes de pedágios, gorjetas, impostos e taxas. | **Tipo:** Numérico (Float/Decimal). |
| `tolls` | Valor total de todos os pedágios pagos na viagem. | **Tipo:** Numérico (Float/Decimal). |
| `bcf` | Valor total coletado na viagem para o Black Car Fund. | **Tipo:** Numérico (Float/Decimal). |
| `sales_tax` | Valor total coletado na viagem para o imposto sobre vendas do estado de NY. | **Tipo:** Numérico (Float/Decimal). |
| `congestion_surcharge` | Valor total coletado na viagem para a sobretaxa de congestionamento do estado de NY. | **Tipo:** Numérico (Float/Decimal). |
| `airport_fee` | Taxa de $2.50 para embarque e desembarque nos aeroportos LaGuardia, Newark e John F. Kennedy. | **Tipo:** Numérico (Float/Decimal). |
| `tips` | Valor total das gorjetas recebidas do passageiro. | **Tipo:** Numérico (Float/Decimal). |
| `driver_pay` | Pagamento total do motorista (sem incluir pedágios ou gorjetas e líquido de comissão, sobretaxas ou impostos). | **Tipo:** Numérico (Float/Decimal). <br> **Regra de Negócio Crítica:** As exclusões devem ser consideradas em qualquer análise de receita. |
| `shared_request_flag` | O passageiro concordou com uma viagem compartilhada/em grupo, independentemente de ter sido pareado? (Y/N) . | **Tipo:** String/Booleano. <br> **Notas:** Mede a **intenção** do passageiro. |
| `shared_match_flag` | O passageiro compartilhou o veículo com outro passageiro que reservou separadamente em algum momento durante a viagem? (Y/N) . | **Tipo:** String/Booleano. <br> **Notas:** Mede a **execução** da viagem compartilhada. |
| `access_a_ride_flag` | A viagem foi administrada em nome da Metropolitan Transportation Authority (MTA)? (Y/N) . | **Tipo:** String/Booleano. |
| `wav_request_flag` | O passageiro solicitou um veículo acessível para cadeira de rodas (WAV)? (Y/N) . | **Tipo:** String/Booleano. <br> **Notas:** Mede a **demanda** por WAV. |
| `wav_match_flag` | A viagem ocorreu em um veículo acessível para cadeira de rodas (WAV)? (Y/N) . | **Tipo:** String/Booleano. <br> **Notas:** Mede a **oferta** de WAV. |
| `cbd_congestion_fee` | Taxa por viagem para a Zona de Alívio de Congestionamento da MTA, a partir de 5 de janeiro de 2025. | **Tipo:** Numérico (Float/Decimal). <br> **Notas:** Relevante para a evolução do esquema (schema evolution). |

---

## Considerações para Arquitetura de Dados

1.  **Modelo Dimensional (Star Schema):** Este conjunto de dados é ideal para um modelo estrela com uma tabela `Fact_Trips_HVFHS` central. As dimensões devem incluir:
    * `Dim_Company` (a partir de `hvfhs_license_num`)
    * `Dim_Base` (a partir de `dispatching_base_num` e `originating_base_num`)
    * `Dim_Location` (a partir de `PULocationID` e `DOLocationID`)
    * `Dim_Date` (a partir de `request_datetime` e `pickup_datetime`)

2.  **Análise Comportamental (Flags):** A presença de pares de flags (`shared_request`/`shared_match` e `wav_request`/`wav_match`) é uma característica rica deste conjunto de dados. Eles permitem análises complexas de funil, como:
    * **Viagens Compartilhadas:** Comparar a demanda (quem solicitou) com a taxa de sucesso de pareamento.
    * **Acessibilidade:** Comparar a demanda por veículos acessíveis com a real oferta e atendimento.

3.  **Particionamento:** A tabela de fatos deve ser particionada por `pickup_datetime` (por exemplo, por ano/mês) para otimizar a performance de consultas e o gerenciamento de dados.

4.  **Evolução do Esquema:** O modelo de dados deve ser flexível para acomodar a introdução de novas taxas, como a `cbd_congestion_fee` a partir de 2025. Dados anteriores a esta data terão valores nulos para este campo.

5.  **Regras de Negócio no ETL:** O pipeline de transformação deve:
    * Normalizar os valores `Y`/`N` para campos booleanos.
    * Documentar claramente a definição do campo `driver_pay`, explicitando o que está incluído e excluído no cálculo.
    * Lidar com a lógica do campo `on_scene_datetime`, que é específico para um tipo de veículo.