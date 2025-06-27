---
title: "🚀 From GCS to BigQuery: Building Scalable ETL Pipelines with Apache Airflow"
date: 2024-06-09
hero: "/images/airflow-gcs-bq.png"
excerpt: "A practical guide to building robust, scalable ETL pipelines using Apache Airflow, Google Cloud Storage, and BigQuery. Learn dynamic task generation, error handling, and best practices for real-world data engineering."
authors:
  - Vedant
---

# 🚀 From GCS to BigQuery: Building Scalable ETL Pipelines with Apache Airflow

As a data engineer, I've learned that the magic isn't just in processing data—it's in orchestrating complex workflows that can handle real-world chaos while maintaining reliability and scalability. Today, I want to share a practical Apache Airflow DAG that demonstrates how to build a robust ETL pipeline for loading and transforming global health data.

## 🎯 The Challenge

Picture this: You have a CSV file containing global health data sitting in Google Cloud Storage, and you need to:
- Load it into BigQuery for analysis
- Create country-specific tables for regional teams
- Generate reporting views with cleaned column names
- Handle failures gracefully and monitor the entire process

This is exactly the scenario I tackled, and here's how Airflow made it elegant and maintainable.

## 🏗️ Architecture Overview

{{< mermaid >}}
graph TD
    A[GCS Bucket] -->|CSV File| B[File Existence Check]
    B --> C[Load to BigQuery Staging]
    C --> D[Create Country Tables]
    D --> E[Create Reporting Views]
    E --> F[Success Notification]
    
    subgraph "BigQuery Datasets"
        G[staging_dataset]
        H[transform_dataset] 
        I[reporting_dataset]
    end
    
    C --> G
    D --> H
    E --> I
{{< /mermaid >}}

## 💡 The Solution: A Multi-Stage ETL DAG##
# Apache Airflow ETL Pipeline: GCS to BigQuery

## The Complete DAG Implementation

```python
from datetime import datetime
from airflow import DAG
from airflow.providers.google.cloud.sensors.gcs import GCSObjectExistenceSensor
from airflow.providers.google.cloud.transfers.gcs_to_bigquery import GCSToBigQueryOperator
from airflow.providers.google.cloud.operators.bigquery import BigQueryInsertJobOperator
from airflow.operators.dummy_operator import DummyOperator

# DAG default arguments
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
}

# Define project, dataset, and table details
project_id = 'tt-dev-02'
dataset_id = 'staging_dataset'
transform_dataset_id = 'transform_dataset'
reporting_dataset_id = 'reporting_dataset'
source_table = f'{project_id}.{dataset_id}.global_data'
countries = ['USA', 'India', 'Germany', 'Japan', 'France', 'Canada', 'Italy']

# DAG definition
with DAG(
    dag_id='load_and_transform_view',
    default_args=default_args,
    description='Load a CSV file from GCS to BigQuery and create country-specific tables',
    schedule_interval=None,  # Manual trigger
    start_date=datetime(2024, 1, 1),
    catchup=False,
    tags=['bigquery', 'gcs', 'csv'],
) as dag:

    # Task 1: Check if the file exists in GCS
    check_file_exists = GCSObjectExistenceSensor(
        task_id='check_file_exists',
        bucket='bkt-src-global-data',
        object='global_health_data.csv',
        timeout=300,
        poke_interval=30,
        mode='poke',
    )

    # Task 2: Load CSV from GCS to BigQuery
    load_csv_to_bigquery = GCSToBigQueryOperator(
        task_id='load_csv_to_bq',
        bucket='bkt-src-global-data',
        source_objects=['global_health_data.csv'],
        destination_project_dataset_table=source_table,
        source_format='CSV',
        allow_jagged_rows=True,
        ignore_unknown_values=True,
        write_disposition='WRITE_TRUNCATE',
        skip_leading_rows=1,
        field_delimiter=',',
        autodetect=True,
    )

    # Tasks 3: Create country-specific tables and views
    create_table_tasks = []
    create_view_tasks = []
    
    for country in countries:
        # Create country-specific table
        create_table_task = BigQueryInsertJobOperator(
            task_id=f'create_table_{country.lower()}',
            configuration={
                "query": {
                    "query": f"""
                        CREATE OR REPLACE TABLE `{project_id}.{transform_dataset_id}.{country.lower()}_table` AS
                        SELECT *
                        FROM `{source_table}`
                        WHERE country = '{country}'
                    """,
                    "useLegacySql": False,
                }
            },
        )

        # Create reporting view with cleaned column names
        create_view_task = BigQueryInsertJobOperator(
            task_id=f'create_view_{country.lower()}_table',
            configuration={
                "query": {
                    "query": f"""
                        CREATE OR REPLACE VIEW `{project_id}.{reporting_dataset_id}.{country.lower()}_view` AS
                        SELECT 
                            `Year` AS `year`, 
                            `Disease Name` AS `disease_name`, 
                            `Disease Category` AS `disease_category`, 
                            `Prevalence Rate` AS `prevalence_rate`, 
                            `Incidence Rate` AS `incidence_rate`
                        FROM `{project_id}.{transform_dataset_id}.{country.lower()}_table`
                        WHERE `Availability of Vaccines Treatment` = False
                    """,
                    "useLegacySql": False,
                }
            },
        )

        # Set task dependencies
        create_table_task.set_upstream(load_csv_to_bigquery)
        create_view_task.set_upstream(create_table_task)
        
        create_table_tasks.append(create_table_task)
        create_view_tasks.append(create_view_task)

    # Success task
    success_task = DummyOperator(task_id='success_task')

    # Define final dependencies
    check_file_exists >> load_csv_to_bigquery
    for create_table_task, create_view_task in zip(create_table_tasks, create_view_tasks):
        create_table_task >> create_view_task >> success_task
```

## Key Components Breakdown

### 1. **File Validation with GCS Sensor**
```python
check_file_exists = GCSObjectExistenceSensor(
    task_id='check_file_exists',
    bucket='bkt-src-global-data',
    object='global_health_data.csv',
    timeout=300,
    poke_interval=30,
    mode='poke',
)
```
This sensor ensures the source file exists before proceeding, preventing downstream failures.

### 2. **Automated Schema Detection**
```python
load_csv_to_bigquery = GCSToBigQueryOperator(
    # ... other parameters ...
    autodetect=True,  # Automatically detect schema
    skip_leading_rows=1,  # Skip CSV header
    allow_jagged_rows=True,  # Handle inconsistent row lengths
)
```

### 3. **Dynamic Task Generation**
The DAG dynamically creates tasks for each country, making it easily scalable:

```python
for country in countries:
    create_table_task = BigQueryInsertJobOperator(...)
    create_view_task = BigQueryInsertJobOperator(...)
```

## Data Flow Visualization

{{< mermaid >}}
graph TD
    A[GCS Bucket: global_health_data.csv] --> B[BigQuery Staging: staging_dataset.global_data]
    B --> C[Country Tables: transform_dataset.COUNTRY_table]
    C --> D[Reporting Views: reporting_dataset.COUNTRY_view]
{{< /mermaid >}}
## Configuration Highlights

### Multi-Dataset Architecture
- **staging_dataset**: Raw data from GCS
- **transform_dataset**: Country-specific filtered tables  
- **reporting_dataset**: Clean views for end users

### Error Handling
- File existence validation before processing
- `allow_jagged_rows=True` for handling inconsistent CSV data
- `ignore_unknown_values=True` for data quality resilience

### Performance Optimizations
- `WRITE_TRUNCATE` for efficient full refreshes
- Parallel execution of country-specific transformations
- Standard SQL for better performance than Legacy SQL


 🔧 What Makes This Pipeline Special

### 1. **Robust File Handling**
The `GCSObjectExistenceSensor` is a game-changer. Instead of blindly trying to process files, we verify they exist first. This prevents cascade failures and provides clear error messages when source data is missing.

### 2. **Dynamic Task Generation**
Rather than hardcoding tasks for each country, the DAG dynamically generates them. Want to add Brazil to your analysis? Just add it to the `countries` list. The pipeline scales automatically.

### 3. **Three-Layer Data Architecture**
```
{{ <mermaid> }}
graph TD
    A[staging_dataset 📁] -->|Raw data landing zone| B[transform_dataset 📁]
    B -->|Business logic applied| C[reporting_dataset 📁]
    C -->|Clean, user-friendly views| D[End Users 👥]
{{ </mermaid> }}
```

This separation of concerns makes debugging easier and provides clear data lineage.

### 4. **Column Renaming Magic**
The reporting views transform ugly column names like `Disease Name` into clean, snake_case versions like `disease_name`. Your analysts will thank you.

## 📊 Pipeline Monitoring Dashboard

When you run this DAG, you'll see something like this in the Airflow UI:

```
✅ check_file_exists
✅ load_csv_to_bq
├── ✅ create_table_usa
│   └── ✅ create_view_usa_table
├── ✅ create_table_india  
│   └── ✅ create_view_india_table
├── ✅ create_table_germany
│   └── ✅ create_view_germany_table
└── ... (and so on)
✅ success_task
```

## 🚨 Lessons Learned

### 1. **Always Validate Your Sources**
The GCS sensor saved me countless hours of debugging. File missing? The pipeline stops gracefully instead of failing mysteriously downstream.

### 2. **Schema Autodetection is Your Friend**
With `autodetect=True`, BigQuery handles schema inference automatically. No more manually defining 50+ columns!

### 3. **Parallel Processing Wins**
By creating country tables in parallel, what used to take 30 minutes now completes in under 5 minutes.

### 4. **Views > Tables for Reporting**
The final reporting layer uses views instead of materialized tables. This keeps storage costs low while providing fresh data for dashboards.

## 🎯 Real-World Applications

This pattern works beautifully for:
- **Multi-tenant SaaS applications** (replace countries with client IDs)
- **Time-series data partitioning** (replace countries with date ranges)
- **A/B testing pipelines** (replace countries with test groups)
- **Regional compliance requirements** (data sovereignty by country)

## 🔮 What's Next?

Here are some enhancements I'm considering:
- Adding data quality checks with Great Expectations
- Implementing incremental loading for large datasets
- Adding Slack notifications for pipeline status
- Creating automated data documentation with dbt

## 💭 Final Thoughts

This DAG represents more than just code—it's a blueprint for scalable, maintainable data pipelines. The combination of Airflow's orchestration capabilities with BigQuery's processing power creates a robust foundation for any data team.

The beauty lies in the details: file validation, dynamic task generation, clean data architecture, and parallel processing. These aren't just best practices—they're battle-tested solutions that scale with your business.

## 🤝 Let's Connect!

If you're building similar pipelines or have questions about this approach, I'd love to hear from you! Data engineering is a journey best traveled with fellow practitioners who understand the joy of a well-orchestrated pipeline and the pain of debugging a 3 AM failure.

What's your go-to orchestration pattern? Have you tried similar multi-stage transformations? Share your experiences in the comments!

---

*Want to see more real-world data engineering content? Follow me for practical tutorials, lessons learned, and honest takes on the tools that actually work in production.* 