# Trabalho 2 - Apache Spark + Delta Lake + MinIO + SQL Server

Projeto desenvolvido para a disciplina de Engenharia de Dados. Implementa um pipeline completo de dados utilizando Apache Spark com Delta Lake, MinIO (Object Storage) e SQL Server 2025.

## Visão Geral

```
┌─────────────────┐     ┌──────────────────┐     ┌───────────────────┐
│   SQL Server    │────▶│   MinIO (S3)     │────▶│   MinIO (S3)      │
│   2025 Dev      │     │   landing-zone/  │     │   bronze/         │
│                 │     │   (CSVs)         │     │   (Delta Tables)  │
│  ParlamentarDB  │     │                  │     │                   │
│   5 tabelas     │     │   1 CSV/tabela   │     │   INSERT/UPDATE   │
│                 │     │                  │     │   DELETE/HISTORY  │
└─────────────────┘     └──────────────────┘     └───────────────────┘
     Notebook 00             Notebook 01            Notebooks 02/03
     (Setup)                 (Extração)             (Delta + DML)
```

## Fonte dos Dados

[Brazilian Parliamentary Expenses Datasets (BPED)](https://www.kaggle.com/datasets/joaopauloschiavon/brazilian-parliamentary-expenses-datasets-bped) — Kaggle.

Dados baseados na **CEAP** (Cota para o Exercício da Atividade Parlamentar) da Câmara dos Deputados do Brasil. O dataset original contém registros de reembolsos nas categorias: Alimentação, Combustíveis, Locação de Veículos e Telefonia. Para fins didáticos, utilizamos uma amostra normalizada de 20–30 registros por tabela.

**Tabelas:**

| Tabela              | Registros | Descrição                              |
|---------------------|-----------|----------------------------------------|
| `partidos`          | 12        | Partidos políticos                     |
| `parlamentares`     | 25        | Deputados federais                     |
| `categorias_despesa`| 8         | Tipos de despesa do CEAP               |
| `fornecedores`      | 20        | Empresas/pessoas que prestaram serviços|
| `despesas`          | 30        | Registros de reembolso do CEAP         |

## Pré-requisitos

- **Linux** (Ubuntu 24.04 ou WSL do Windows 11)
- **Docker** e **Docker Compose** v2+
- **Python 3.11**
- **Java 11** (OpenJDK)
- **UV** (gerenciador de pacotes Python) — [instalação](https://github.com/astral-sh/uv)
- **ODBC Driver 18 for SQL Server**

## Setup do Ambiente

### 1. Subir os Containers (SQL Server + MinIO)

```bash
docker compose up -d
```

| Container       | Imagem                                       | Portas         |
|-----------------|----------------------------------------------|----------------|
| sqlserver-2025  | `mcr.microsoft.com/mssql/server:2025-latest` | `1433`         |
| minio           | `minio/minio:RELEASE.2025-02-03T21-03-04Z`   | `9020`, `9021` |

| Serviço    | Usuário      | Senha             |
|------------|--------------|-------------------|
| SQL Server | `sa`         | `SqlServer@2025!` |
| MinIO      | `minioadmin` | `minioadmin`      |

Console MinIO: http://localhost:9021

### 2. Configurar Variáveis de Ambiente

```bash
cp .env.example .env
```

### 3. Configurar o Ambiente Python

```bash
uv venv
source .venv/bin/activate
uv sync
```

### 4. Instalar ODBC Driver (Ubuntu 24.04)

```bash
sudo apt install -y unixodbc-dev
curl https://packages.microsoft.com/keys/microsoft.asc | sudo tee /etc/apt/trusted.gpg.d/microsoft.asc
sudo curl https://packages.microsoft.com/config/ubuntu/24.04/prod.list | sudo tee /etc/apt/sources.list.d/mssql-release.list
sudo apt update
sudo ACCEPT_EULA=Y apt install -y msodbcsql18
```

Validar: `odbcinst -q -d` deve retornar `[ODBC Driver 18 for SQL Server]`.

## Executando o Projeto

Execute os notebooks **em ordem**:

| # | Notebook | Descrição |
|---|----------|-----------|
| 0 | `00_setup_sqlserver.ipynb` | Cria `ParlamentarDB` e carrega as 5 tabelas |
| 1 | `01_sqlserver_to_minio_csv.ipynb` | Extrai todas as tabelas → CSV no MinIO (`landing-zone`) |
| 2 | `02_csv_to_delta.ipynb` | Lê CSVs do MinIO e converte para Delta Lake (`bronze`) |
| 3 | `03_dml_delta.ipynb` | Executa DML (INSERT, UPDATE, DELETE), HISTORY e TIME TRAVEL |

> Selecione o kernel `.venv` no Jupyter antes de executar.

## Estrutura do Projeto

```
sqlserver-minio-spark-delta/
├── docker-compose.yml
├── .env.example
├── pyproject.toml
├── .python-version
├── data/
│   ├── partidos.csv
│   ├── parlamentares.csv
│   ├── categorias_despesa.csv
│   ├── fornecedores.csv
│   └── despesas.csv
└── notebook/
    ├── 00_setup_sqlserver.ipynb
    ├── 01_sqlserver_to_minio_csv.ipynb
    ├── 02_csv_to_delta.ipynb
    └── 03_dml_delta.ipynb
```

## Tecnologias Utilizadas

- **Apache Spark 3.5.3** (PySpark) — Motor de processamento distribuído
- **Delta Lake 3.2.0** — Formato de armazenamento com suporte ACID
- **MinIO** — Object Storage compatível com S3
- **SQL Server 2025** — Banco de dados relacional (Developer Edition)
- **Docker Compose** — Orquestração de containers
- **Python 3.11** com UV

## Conceitos Demonstrados

- **Extração** de dados de banco relacional (SQL Server) via pyodbc
- **Landing Zone** — armazenamento intermediário em CSV no MinIO
- **Delta Lake** — formato ACID com versionamento
- **Arquitetura Medalhão** — Landing Zone → Bronze
- **DML** em tabelas Delta (INSERT, UPDATE, DELETE via SQL e DeltaTable API)
- **History** — log completo de transações (auditoria de dados públicos)
- **Time Travel** — leitura de versões anteriores (`versionAsOf`)
- **Tabelas Gerenciadas vs Não Gerenciadas** — discussão em sala

## Tabelas Gerenciadas vs Não Gerenciadas

| Característica | Gerenciada (Interna) | Não Gerenciada (Externa) |
|----------------|---------------------|--------------------------|
| Metadados      | Spark gerencia      | Spark gerencia           |
| Dados          | Spark gerencia      | Usuário gerencia         |
| DROP TABLE     | Remove dados e metadados | Remove apenas metadados |
| Criação        | A partir de DataFrame | A partir de arquivo externo |

Neste projeto utilizamos **tabelas não gerenciadas** (externas), pois os dados ficam armazenados no MinIO e o Spark apenas aponta para o caminho `s3a://`. Se executarmos `DROP TABLE`, os arquivos Delta no MinIO são preservados.

## Referências

- Projeto de referência: [jlsilva01/spark-delta-minio-sqlserver](https://github.com/jlsilva01/spark-delta-minio-sqlserver)
- Dataset: [Brazilian Parliamentary Expenses (BPED) — Kaggle](https://www.kaggle.com/datasets/joaopauloschiavon/brazilian-parliamentary-expenses-datasets-bped)
- [Delta Lake - Documentação](https://docs.delta.io/latest/index.html)
- [MinIO - Documentação](https://min.io/docs/minio/linux/index.html)
- [CEAP - Câmara dos Deputados](https://www2.camara.leg.br/transparencia/acesso-a-informacao/copy_of_aplicacoes-para-dispositivos-moveis/ceap-cota-para-o-exercicio-da-atividade-parlamentar)
