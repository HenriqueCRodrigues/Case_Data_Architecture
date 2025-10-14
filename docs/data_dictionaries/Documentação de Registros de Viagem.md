# Documentação de Registros de Viagem da NYC TLC

Este documento serve como guia de referência técnica para o design e implementação de uma arquitetura de dados que irá processar, armazenar e analisar os conjuntos de dados de registros de viagem da New York City Taxi and Limousine Commission (TLC)

## 1. Visão Geral e Fontes de Dados

A TLC é a agência que licencia e regulamenta os serviços de táxi e veículos de aluguel (FHV) em Nova York A comissão coleta registros de cada viagem concluída

- **Origem dos Dados:**
    - **Táxis (Amarelos e Verdes):** Os dados são recebidos de Provedores de Serviços de Tecnologia (TSPs) que gerenciam os taxímetros eletrônicos
    - **Veículos de Aluguel (FHV):** Os dados são recebidos das bases que despacharam a viagem, incluindo aplicativos, serviços de "black car", limusines, etc.
- **Granularidade:** Em cada conjunto de dados, uma linha representa uma única viagem realizada por um veículo licenciado pela TLC
- **Qualidade dos Dados:** Os dados de viagem não são criados pela TLC, e a agência não pode garantir sua precisão. Esta é uma premissa importante para a camada de ingestão e limpeza de dados (Data Cleansing).

## 2. Aquisição e Acesso aos Dados (ETL/ELT Pipeline)

Existem duas fontes principais para a extração dos dados. A escolha impacta a estratégia de ingestão.

### 2.1. Website da TLC
- **Endereço:** `https://www1.nyc.gov/site/tlc/about/tic-trip-record-data.page`.
- **Formato:** Os dados são separados por ano, mês e tipo de serviço (yellow/green/FHV) e disponibilizados como arquivos CSV.
- **Volume:** Os arquivos mensais podem conter milhões de registros, ocupando um espaço considerável em disco.
- **Frequência de Atualização:** A TLC atualiza os registros a cada seis meses. Dados de Janeiro a Junho são esperados até o final de Agosto, e dados de Julho a Dezembro até o final de Fevereiro.
- **Estratégia Recomendada:** Ideal para cargas em lote (batch processing) de dados históricos e atualizações semestrais.

### 2.2. Portal NYC Open Data
- **Endereço:** `https://opendata.cityofnewyork.us/`.
- **Funcionalidades:**
    - Permite filtrar os dados antes do download (ex: por data, tarifa, local), otimizando a ingestão.
    - Suporta múltiplos formatos de exportação (CSV, JSON, XML, etc.).
    - Oferece acesso via API para ingestão programática e contínua.
- **Metadados:** É fundamental consultar os metadados e o dicionário de dados de cada dataset para entender o significado de cada coluna e a data da última atualização.
- **Estratégia Recomendada:** Ideal para cargas de dados mais seletivas, pipelines de streaming (via API) ou quando diferentes formatos de arquivo são necessários.

## 3. Linha do Tempo e Evolução do Esquema (Schema Evolution)

A estrutura dos dados, especialmente para FHVs, mudou ao longo do tempo. O data warehouse deve ser projetado para lidar com essas variações.

- **2009:** Início do recebimento de dados de táxis
- **2013:** Adição dos Táxis Verdes (Green Taxis).
- **2015:**
    - Dados de táxis (amarelos e verdes) são publicados no Open Data.
    - Início do recebimento de dados de FHV, contendo apenas: `dispatching base`, `pickup datetime` e `pickup location`.
- **2017:** Dados de FHV são enriquecidos para incluir `drop-off datetime` e `drop-off location`. Também se inicia a coleta de dados sobre corridas compartilhadas.
- **2019:**
    - Criação da licença **High Volume For Hire Services (HVFHS)** para empresas com mais de 10.000 viagens/dia (Uber, Lyft, Via, Juno).
    - Esta nova classe possui requisitos de reporte de dados mais rigorosos, buscando paridade com os dados de táxis. Um campo de identificação da licença de alto volume foi adicionado.

## 4. Modelagem de Dados: Entidades e Dimensões

### 4.1. Tabelas de Fatos (Fact Tables)
Haverá tabelas de fatos distintas para cada tipo de serviço, devido às diferenças nos esquemas.

- **Fato_Viagens_Taxi (Amarelo e Verde):**
    - **Descrição:** Registros detalhados de viagens com taxímetro. Os táxis amarelos são os únicos autorizados a pegar passageiros em qualquer rua dos cinco distritos. Os táxis verdes são restritos a áreas específicas.
    - **Campos Chave:** Datas/horas de embarque e desembarque, IDs de localização, distância, valores detalhados da tarifa, tipo de pagamento, contagem de passageiros 70.

- **Fato_Viagens_FHV:**
    - **Descrição:** Registros de bases de FHV que não se enquadram como alto volume. O esquema é mais simples e varia por ano.
    - **Campos Chave:** Número da base de despacho, datas/horas e IDs de localização de embarque/desembarque.

- **Fato_Viagens_HVFHS:**
    - **Descrição:** Registros de empresas de alto volume (Uber, Lyft, etc.) a partir de Fev/2019.
    - **Campos Chave:** Número da licença HVFHS, número da base, datas/horas de solicitação e viagem, IDs de localização, milhas e tempo da viagem, valores da tarifa, gorjetas, pagamento do motorista e flags de viagem compartilhada/acessível.

### 4.2. Tabelas de Dimensão (Dimension Tables)

- **Dim_Localizacao (Taxi Zones):**
    - **Propósito:** Mapear os IDs de localização (`PULocationID`, `DOLocationID`) para informações geográficas.
    - **Chave:** `LocationID` (valores de 1 a 263).
    - **Atributos:** Bairro, Distrito, Geometria (Shapefile).
    - **Fonte:** Disponível como tabela de consulta (CSV) ou Shapefile no site da TLC e no portal Open Data. As zonas são baseadas nas Neighborhood Tabulation Areas (NTAs).

- **Dim_Bases_FHV:**
    - **Propósito:** Identificar a empresa e a base que despachou uma viagem de FHV/HVFHS.
    - **Chave:** `dispatching_base_num` (ex: "B02914").
    - **Atributos:** Nome da Base, Número da Licença HVFHS, Nome da Empresa Afiliada (ex: "Juno").
    - **Fonte:** O guia fornece uma tabela de mapeamento crucial para as empresas de HVFHS, pois o nome da base não é o nome comercial da empresa.

## 5. Regras de Negócio Críticas

- **Viagens Compartilhadas (`SR_Flag`):**
    - Este campo, presente nos registros de FHV de 2018, indica se a viagem fez parte de uma corrida compartilhada (ex: Uber Pool). O valor é `1` para compartilhada e nulo para não compartilhada.
    - **Regra Geral:** A flag é `1` apenas se a viagem foi solicitada como compartilhada **E** efetivamente pareada com outro passageiro.
    - **Exceção Importante:** A Lyft (hvfhs_license_num='HV0005') define a flag como `1` se a viagem foi **solicitada** como compartilhada, mesmo que não tenha sido pareada. Isso pode levar a uma contagem excessiva de viagens compartilhadas bem-sucedidas da Lyft.
    - A Juno não oferece viagens compartilhadas.

- **Identificação de Empresas HVFHS:** Para identificar corretamente a empresa por trás de uma viagem de alto volume (ex: Uber, Lyft), é **obrigatório** usar a tabela de junção que mapeia o `dispatching_base_num` para a empresa afiliada, conforme detalhado no guia.

## 6. Contato e Recursos Adicionais

- **Dúvidas sobre os dados:** `research@flc.nyc.gov`.
- **Blog da TLC com projetos e análises:** `https://medium.com/@NYCTLC`.