# Análise de Dicionário de Dados: FHV Trip Records

**Fonte:** Data Dictionary - FHV Trip Records
**Data do Documento:** 18 de Março de 2025 

Este documento descreve a estrutura e os campos dos dados de viagem para veículos de aluguel (For-Hire Vehicles - FHV). É crucial notar que, a partir de 2019, as empresas de FHV de alto volume (como Uber, Lyft, etc.) foram separadas em um conjunto de dados distinto. Portanto, este dicionário de dados se aplica principalmente a viagens de bases que não são de alto volume a partir dessa data.

---

## Tabela de Campos e Dicionário de Dados

Cada linha neste conjunto de dados representa uma única viagem em um FHV.

| Nome do Campo | Descrição | Valores e Notas para Modelagem |
|---|---|---|
| `dispatching_base_num` | O número da licença da base da TLC que despachou a viagem. | **Tipo:** String. <br> **Notas:** Chave primária para junção com uma tabela de dimensão `Dim_Base` para obter mais detalhes sobre a base de despacho. |
| `pickup_datetime` | A data e a hora do embarque da viagem. | **Tipo:** Timestamp. <br> **Notas:** Campo ideal para o particionamento da tabela de fatos, por exemplo, por ano e mês. |
| `dropOff_datetime` | A data e a hora do desembarque da viagem. | **Tipo:** Timestamp. |
| `PUlocationID` | A Zona de Táxi da TLC onde a viagem começou. | **Tipo:** Inteiro. <br> **Notas:** Chave estrangeira para a dimensão `Dim_Location`. |
| `DOlocationID` | A Zona de Táxi da TLC onde a viagem terminou. | **Tipo:** Inteiro. <br> **Notas:** Chave estrangeira para a dimensão `Dim_Location`. |
| `SR_Flag` | Indica se a viagem fez parte de uma corrida compartilhada (ex: Uber Pool, Lyft Line). | **Tipo:** Inteiro/Nulo. <br> **Valores:** `1` para viagens compartilhadas; `Nulo` para viagens não compartilhadas. <br> **Regra de Negócio Crítica:** <br> * **Geral:** A flag é marcada (`1`) apenas para viagens que foram solicitadas **E** pareadas com outra solicitação de viagem compartilhada. <br> * **Exceção (Lyft):** A Lyft (números de licença de base B02510 + B02844) marca a flag (`1`) também para viagens em que uma corrida compartilhada foi solicitada, mas nunca foi pareada com sucesso com outro passageiro. Isso deve ser tratado no pipeline de ETL para evitar uma contagem excessiva de viagens compartilhadas bem-sucedidas. |
| `Affiliated_base_numb` | O número da base com a qual o veículo é afiliado. | **Tipo:** String. <br> **Notas:** Este campo deve ser fornecido mesmo que a base afiliada seja a mesma que a base de despacho. Pode ser usado para enriquecer a dimensão `Dim_Vehicle` ou `Dim_Base`. |

---

## Considerações para Arquitetura de Dados

1.  **Modelo Dimensional (Star Schema):** Os dados são bem estruturados para um modelo estrela, com uma tabela `Fact_Trips_FHV` central. As dimensões associadas seriam:
    * `Dim_Base` (usando `dispatching_base_num` e `Affiliated_base_numb` como chaves).
    * `Dim_Location` (usando `PUlocationID` e `DOlocationID`).
    * `Dim_Date` (gerada a partir de `pickup_datetime` e `dropOff_datetime`).

2.  **Evolução do Esquema e Contexto dos Dados:** A mudança mais significativa foi a separação dos dados de alto volume em 2019. Ao analisar dados que abrangem esse período, é essencial filtrar ou segmentar as viagens para evitar comparações incorretas entre os períodos pré e pós-2019. Este conjunto de dados representa um segmento de mercado diferente após essa data.

3.  **Regras de Negócio no ETL/ELT:** O pipeline de transformação de dados deve conter uma lógica explícita e bem documentada para lidar com o campo `SR_Flag`. Deve-se criar uma coluna adicional (ex: `is_successfully_shared_trip`) que aplique a regra de exceção da Lyft para permitir análises precisas.

4.  **Chaves e Junções:** O campo `dispatching_base_num` é a chave principal para entender a origem da viagem. A junção com uma tabela de dimensões de bases licenciadas pela TLC é fundamental para enriquecer os dados.

5.  **Qualidade de Dados:** O campo `SR_Flag` é complexo e propenso a má interpretação. A documentação do data warehouse e dos relatórios derivados deve destacar claramente a lógica aplicada para a contagem de viagens compartilhadas, especialmente a distinção entre "solicitada" e "efetivamente compartilhada".