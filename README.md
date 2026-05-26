# InduSense

**A machine learning data pipeline for industrial sensor and incident data processing**

InduSense is a PostgreSQL-based data ingestion and transformation platform designed to process heterogeneous sensor data (temperature, pressure) and incident reports from industrial machines. The project implements a Bronze-Silver-Gold data warehouse architecture to ensure data quality, traceability, and machine learning readiness.

## Overview

### Key Constraints

Data exploration identified three critical constraints:
- The three data sources have heterogeneous separators and formats
- Machine identifiers require normalization before any join operations
- The common exploitable time window for sensor/incident cross-referencing spans **2025-08-26 to 2026-02-25**, with **15 machines** after harmonization

### Design Goals

Rather than directly loading CSV files into a final table, the architecture prepares a relational model that enables:
- Raw file landing with full traceability
- Processing lineage and rejection tracking
- Normalized Silver-layer tables
- Production-ready Gold dataset with hourly granularity, protected against data leakage

## Architecture

### Three-Layer Data Warehouse

#### 🔴 Bronze Layer
Preserves raw data as-is with:
- Raw row content from source files
- Source file lineage tracking
- Ingestion status and batch information
- Data quality issue flags

#### 🟡 Silver Layer
Normalized, deduplicated data with:
- Standardized machine IDs, timestamps, and data types
- Unified sensor readings (temperature + pressure) per machine per hour
- Incident records with pseudonymized operator information
- Isolated data quality issues and invalid records

#### 🟢 Gold Layer
Machine learning-ready dataset with:
- Hourly features per machine (sliding window aggregations)
- Explanatory variables and target labels
- Temporal train/validation/test split definitions (preventing data leakage)
- Ready for model training and evaluation

## Project Structure

```
indusense_ml/
├── src/indusense/
│   ├── core/              # Core utilities (logging, settings)
│   ├── db/                # Database models and session management
│   ├── pipeline/          # ETL pipeline stages (bronze, silver, gold)
│   ├── processing/        # Data processing and reporting logic
│   ├── schemas/           # Pydantic validation schemas
│   └── weather/           # External data fetchers (weather, etc.)
├── alembic/               # Database migrations
├── artifacts/             # Data ingestion reports and artifacts
├── logs/                  # Application logs
├── b1_explore.ipynb       # Exploratory data analysis notebook
├── main.py                # Pipeline entry point
├── pyproject.toml         # Project dependencies and metadata
└── alembic.ini            # Alembic configuration
```

## Setup

### Prerequisites
- Python 3.13+
- PostgreSQL 13+
- pip or similar package manager

### Installation

1. **Clone and navigate to the project:**
   ```bash
   cd indusense_ml
   ```

2. **Create and activate a virtual environment:**
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies:**
   ```bash
   pip install -e .
   ```

4. **Configure environment variables:**
   Create a `.env` file in the project root:
   ```
   DATABASE_URL=postgresql+psycopg://user:password@localhost/indusense_ml
   LOG_LEVEL=INFO
   ```

5. **Run database migrations:**
   ```bash
   alembic upgrade head
   ```

## Usage

### Running the Full Pipeline

```bash
python main.py
```

This executes the complete ETL pipeline:
1. **Bronze ingestion**: Loads raw data from source files
2. **Silver transformation**: Normalizes and deduplicates data
3. **Gold aggregation**: Creates machine learning features

### Pipeline Components

- **Bronze Pipeline** (`src/indusense/pipeline/bronze.py`): Raw data ingestion with lineage tracking
- **Silver Pipeline** (`src/indusense/pipeline/silver.py`): Data normalization and deduplication
- **Gold Pipeline** (`src/indusense/pipeline/gold.py`): Feature engineering and dataset preparation
- **Data Ingestion** (`src/indusense/processing/ingestion.py`): Core ingestion logic
- **Reporting** (`src/indusense/processing/reporting.py`): Data quality and processing reports

## Technologies

- **SQLAlchemy** (2.0+): ORM for database operations
- **Alembic**: Database schema versioning and migrations
- **Pydantic**: Data validation and schema definition
- **Pandas**: Data manipulation and analysis
- **PostgreSQL**: Relational database backend
- **Loguru**: Advanced logging framework
- **Python-dotenv**: Environment configuration management

## Data Model

### Entity Relationship Overview

```mermaid
erDiagram
    INGESTION_BATCH ||--o{ BRONZE_TEMPERATURE_RAW : loads
    INGESTION_BATCH ||--o{ BRONZE_PRESSURE_RAW : loads
    INGESTION_BATCH ||--o{ BRONZE_INCIDENT_RAW : loads
    INGESTION_BATCH ||--o{ DATA_QUALITY_ISSUE : produces

    MACHINE ||--o{ SILVER_SENSOR_READING : emits
    MACHINE ||--o{ SILVER_INCIDENT : experiences
    OPERATOR ||--o{ SILVER_INCIDENT : reports
    MACHINE ||--o{ GOLD_MACHINE_HOURLY_FEATURE : feeds

    INGESTION_BATCH {
        uuid ingestion_batch_id PK
        string source_name
        string source_file
        timestamptz started_at
        timestamptz finished_at
        int rows_read
        int rows_loaded
        int rows_rejected
        string status
    }

    MACHINE {
        bigint machine_id PK
        string machine_code UK
        date commissioning_date
        int max_daily_capacity
        string model
        string production_line
        string location
        string criticality
        boolean is_active
    }

    OPERATOR {
        bigint operator_id PK
        string operator_key UK
        string badge_hash
        boolean is_active
    }

    BRONZE_TEMPERATURE_RAW {
        bigint temperature_raw_id PK
        uuid ingestion_batch_id FK
        int row_number
        string machine_id_raw
        string timestamp_raw
        string temperature_raw
        boolean parse_ok
    }

    BRONZE_PRESSURE_RAW {
        bigint pressure_raw_id PK
        uuid ingestion_batch_id FK
        int row_number
        string machine_id_raw
        string timestamp_raw
        string pressure_raw
        boolean parse_ok
    }

    BRONZE_INCIDENT_RAW {
        bigint incident_raw_id PK
        uuid ingestion_batch_id FK
        int row_number
        string incident_code_raw
        string machine_id_raw
        string operator_name_raw
        string operator_badge_raw
        string occurred_at_raw
        string severity_raw
        boolean parse_ok
    }

    DATA_QUALITY_ISSUE {
        bigint dq_issue_id PK
        uuid ingestion_batch_id FK
        string dataset_name
        string rule_code
        string severity
        string entity_key
        text details
    }

    SILVER_SENSOR_READING {
        bigint sensor_reading_id PK
        bigint machine_id FK
        timestamptz observed_at
        string sensor_type
        numeric sensor_value
        string unit
        boolean is_missing
        boolean is_duplicate
        boolean is_outlier
        uuid ingestion_batch_id FK
    }

    SILVER_INCIDENT {
        bigint incident_id PK
        string incident_code UK
        bigint machine_id FK
        bigint operator_id FK
        timestamptz occurred_at
        smallint severity
        string shift
        text comment
        boolean is_label_event
        uuid ingestion_batch_id FK
    }

    GOLD_MACHINE_HOURLY_FEATURE {
        bigint feature_row_id PK
        bigint machine_id FK
        timestamptz window_start
        timestamptz window_end
        numeric temp_mean_24h
        numeric temp_max_24h
        numeric temp_std_24h
        numeric pressure_mean_24h
        numeric pressure_std_24h
        int incident_count_prev_24h
        int incident_max_severity_prev_24h
        boolean label_failure_next_24h
        string split_set
    }
```

### Data Model Philosophy

- **Bronze**: Preserves source truth, including heterogeneous identifiers, timestamp formats, and rejectable rows, with complete lineage tracking
- **Silver**: Contains cleaned and normalized facts with quality flags useful for analysis and model interpretability
- **Gold**: Temporal feature table with stable granularity, ready for supervised learning models predicting machine failures

## Key Design Decisions

The following architecture choices have been established:

- ✅ Machine identifiers normalized using `machine_id` (PK) and `machine_code` (UK)
- ✅ Gold layer granularity: **hourly** (machine × hour)
- ✅ Prediction horizon: **24-hour failure window**
- ✅ Deduplication and missing value handling rules applied at Silver layer
- ✅ Temporal train/validation/test splits preserved in `split_set` field

## Contributing

To contribute improvements to the pipeline:

1. Update relevant pipeline modules in `src/indusense/pipeline/`
2. Create database migrations with Alembic if schema changes are needed
3. Update test coverage and data quality rules
4. Document changes in appropriate processing/reporting modules

## License

See project documentation for licensing information.
