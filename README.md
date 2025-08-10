### **Desafio Técnico: Pipeline de Análise de Votação por Seção Eleitoral**

**Contexto:**
Sua tarefa é construir um pipeline de dados ponta a ponta para processar os resultados de votação por seção eleitoral (Boletim de Urna) do Tribunal Superior Eleitoral (TSE) do Brasil. O objetivo final é criar uma camada analítica que permita consultas sobre os resultados da eleição de 2022, agregando votos por candidato, cargo, município e estado.

**Requisitos Fundamentais:**

1.  **Reprodutibilidade:** A solução completa deve ser executável com um único comando: `docker-compose up`. Todos os serviços e o pipeline devem estar containerizados.
2.  **Escalabilidade:** A solução deve ser pensada para lidar com um volume de dados considerável (os arquivos de Boletim de Urna são grandes).
3.  **Código Aberto:** Utilize ferramentas open-source.
4.  **Qualidade de Código:** O código deve ser limpo, modular, testável e bem documentado.

---

### **Arquitetura Proposta (Serviços no `docker-compose.yml`)**

A arquitetura de serviços proposta seria:

1.  **Data Lake (MinIO):** Para armazenar os dados brutos (arquivos `.zip` ou `.csv` extraídos).
2.  **Data Warehouse (PostgreSQL):** Para armazenar os dados tratados e modelados para análise.
3.  **Orquestrador (Airflow):** Para agendar, executar e monitorar o pipeline.
4.  **Serviço de Transformação (Recomendado):** Um container para executar as transformações (ex: um serviço Python que o Airflow aciona).

---

### **Etapas do Pipeline (DAG no Airflow)**

#### **1. Ingestão de Dados (Extract)**

*   **Fonte de Dados:** Utilize o dataset de **Boletim de Urna (bweb)** da eleição de 2022, conforme o link: [Resultados 2022 - Boletim de Urna](https://dadosabertos.tse.jus.br/dataset/resultados-2022/resource/40fdcf49-256a-4c81-87cf-711545bd1528).
*   **Tarefa:** Crie uma tarefa no Airflow que:
    1.  Baixe o arquivo `.zip`.
    2.  Descompacte o arquivo. Dentro, você encontrará vários CSVs com os dados de votação.
    3.  Envie os arquivos CSVs brutos para um bucket no MinIO (ex: `raw-data`).
    *   **Observação:** O processo de ingestão deve ser eficiente e não estourar a memória do container.

#### **2. Carga para o Data Warehouse (Load)**

*   **Tarefa:** Crie uma tarefa que leia o CSV da camada `raw-data` do MinIO e o carregue em uma tabela "staging" no PostgreSQL.
*   **Detalhe:** Devido ao tamanho do arquivo, implemente uma **carga incremental (chunking)**. Leia o CSV em pedaços (ex: 100.000 linhas por vez) e insira no PostgreSQL, em vez de carregar tudo de uma vez na memória.

#### **3. Transformação e Modelagem (Transform)**

Use **dbt** para executar as transformações via SQL diretamente no PostgreSQL.

*   **Tarefas de Transformação:**
    1.  **Limpeza e Tipagem (Staging Layer):** Crie modelos dbt que leiam da tabela staging e realizem a limpeza:
        *   Selecione apenas as colunas relevantes (ex: `NR_TURNO`, `SG_UF`, `NM_MUNICIPIO`, `DS_CARGO_PERGUNTA`, `NR_VOTAVEL`, `NM_VOTAVEL`, `QT_VOTOS`).
        *   Corrija os tipos de dados (ex: `QT_VOTOS` para `INTEGER`, etc.).
        *   Filtre os dados para cargos majoritários (ex: Presidente, Governador) para simplificar a análise inicial.

    2.  **Modelagem Dimensional (Analytics Layer):** Crie modelos de dados otimizados para análise. A abordagem aqui será criar uma única e poderosa **tabela fato** denormalizada, ideal para a granularidade dos dados.
        *   `fct_eleicao`: Uma tabela principal com o grão "votos por candidato/municipio".
            *   Modele e defina as colunas da fato de acordo com os dados da fonte
    3.  **Tabelas Agregadas (Marts):** Crie modelos de dados agregados para responder perguntas de forma rápida:
        *   `mart_resultado_geral_br`: Tabela com o total de votos por candidato para o cargo de Presidente. (se possível com o dataset)
        *   `mart_resultado_por_estado`: Total de votos por candidato e por estado para os cargos de Presidente e Governador. (se possível com o dataset)
        *   `mart_resultado_por_municipio`: Total de votos por candidato, por estado e por município para todos os cargos. (se possível com o dataset)

#### **4. Testes de Qualidade de Dados**

*   **Ferramenta:** Utilize os testes nativos do dbt.
*   **Tarefa:** Implemente testes para garantir a integridade dos dados:
    *   Ex: `not_null` nas colunas chave (`municipio`, `cargo`, `numero_votavel`).
    *   Ex: `accepted_values` para a coluna `NR_TURNO` (valores devem ser 1 ou 2).

---

### **Critérios de Avaliação**

*   **Funcionalidade e Desempenho:** O pipeline executa com `docker-compose up`? Ele consegue processar o grande volume de dados do Boletim de Urna de forma eficiente (especialmente na etapa de carga)?
*   **Estrutura do Projeto:** Organização do repositório.
*   **Qualidade do Código dbt:** Os modelos são modulares, eficientes e bem documentados? A modelagem atende ao objetivo analítico?
*   **Orquestração:** A DAG do Airflow está bem estruturada? Ela lida bem com tarefas de longa duração?
*   **Documentação:** O `README.md` está claro e completo?
*   **Conceitos Avançados:** O candidato demonstrou saber lidar com grandes volumes de dados (chunking), modelagem para performance analítica e testes de consistência entre os modelos.
*   **Sobre uso de AI:** O candidato poderá usar a IA que quiser, desde que deixe documentado os códigos que a AI gerou pra ele ou o raciocínio feito para usar a AI, no caso de não geração de código por parte dela. Contudo, o desafio não poderá ser inteiramente desenvolvido com o uso de Agentes, trechos de código são aceitáveis, a implementação inteira, não.