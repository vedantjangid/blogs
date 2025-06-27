---
title: "🚀 From GCS to BigQuery: Building Scalable ETL Pipelines with Apache Airflow"
date: 2025-06-27
hero: "/images/airflow-gcs-bq.png"
excerpt: "A hands-on, real-world guide to building robust, scalable ETL pipelines with Apache Airflow, GCS, and BigQuery. Learn the tricks, the pitfalls, and the magic of orchestration."
authors:
  - Vedant
---

Want to move data from Google Cloud Storage to BigQuery, transform it, and make it analyst-friendly—all in a single, reliable pipeline? Here's how I did it with Airflow, and how you can too.

## 👋 Why This Pipeline?

Let's be honest: data engineering isn't just about crunching numbers. It's about making sure the right data lands in the right place, in the right shape, every single time—even when the world throws you curveballs.

I recently built a pipeline to load global health data from GCS into BigQuery, split it by country, and create clean reporting views. Here's the architecture that made it all work (and saved me from 3 AM alerts):

## 🏗️ How It All Fits Together

<div class="mermaid-container">
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
</div>

## 🧰 Tools & Stack

- **Apache Airflow** 🪁
- **Google Cloud Storage** ☁️
- **BigQuery** 🏢
- **Python** 🐍

## 🎯 The Real-World Challenge

Picture this:

> You've got a CSV of global health data in GCS. Your mission:
> - Load it into BigQuery for analysis
> - Create country-specific tables for regional teams
> - Generate reporting views with clean, analyst-friendly columns
> - Handle failures gracefully and keep tabs on the whole process

Sound familiar? Here's how I tackled it.

## 💡 The Solution: A Multi-Stage ETL DAG

*This isn't just a DAG—it's a battle-tested blueprint for scalable, maintainable data pipelines.*

## 📝 The Complete DAG Implementation

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
project_id = 'atgeir-accelerators'
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

## 🧩 Key Building Blocks (And Why They Matter)

### 1. File Validation with GCS Sensor

> **Pro Tip:** Always check if your source file exists before you start processing. It'll save you hours of debugging!

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

### 2. Automated Schema Detection

> **Why I love this:** With `autodetect=True`, BigQuery figures out the schema for you. No more manually defining 50+ columns!

```python
load_csv_to_bigquery = GCSToBigQueryOperator(
    # ... other parameters ...
    autodetect=True,  # Automatically detect schema
    skip_leading_rows=1,  # Skip CSV header
    allow_jagged_rows=True,  # Handle inconsistent row lengths
)
```

### 3. Dynamic Task Generation

> **Scalability win:** Want to add Brazil? Just add it to the `countries` list. The pipeline scales itself.

```python
for country in countries:
    create_table_task = BigQueryInsertJobOperator(...)
    create_view_task = BigQueryInsertJobOperator(...)
```

## 🔄 Data Flow: See It in Action

<div class="mermaid-container">
{{< mermaid >}}
graph TD
    A[GCS Bucket: global_health_data.csv] --> B[BigQuery Staging: staging_dataset.global_data]
    B --> C[Country Tables: transform_dataset.COUNTRY_table]
    C --> D[Reporting Views: reporting_dataset.COUNTRY_view]
{{< /mermaid >}}
</div>

## ⚙️ Configuration Highlights

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

## 🏆 What Makes This Pipeline Special?

> **Battle-tested:** These aren't just best practices—they're solutions I've used in production, under real pressure.

### 1. Robust File Handling

The `GCSObjectExistenceSensor` is a game-changer. Instead of blindly trying to process files, we verify they exist first. This prevents cascade failures and provides clear error messages when source data is missing.

### 2. Dynamic Task Generation

Rather than hardcoding tasks for each country, the DAG dynamically generates them. Want to add Brazil to your analysis? Just add it to the `countries` list. The pipeline scales automatically.

### 3. Three-Layer Data Architecture

<div class="mermaid-container">
{{< mermaid >}}
graph TD
    A[staging_dataset 📁] -->|Raw data landing zone| B[transform_dataset 📁]
    B -->|Business logic applied| C[reporting_dataset 📁]
    C -->|Clean, user-friendly views| D[End Users 👥]
{{< /mermaid >}}
</div>

> **Why it matters:** This separation of concerns makes debugging easier and provides clear data lineage.

### 4. Column Renaming Magic

The reporting views transform ugly column names like `Disease Name` into clean, snake_case versions like `disease_name`. Your analysts will thank you.

## 📊 Pipeline Monitoring Dashboard

When you run this DAG, you'll see something like this in the Airflow UI:

<div class="mermaid-container">
{{< mermaid >}}
graph TD
    A[✅ check_file_exists] --> B[✅ load_csv_to_bq]
    B --> C1[✅ create_table_usa]
    C1 --> D1[✅ create_view_usa_table]
    B --> C2[✅ create_table_india]
    C2 --> D2[✅ create_view_india_table]
    B --> C3[✅ create_table_germany]
    C3 --> D3[✅ create_view_germany_table]
    B --> Cn[... More create_table_x]
    Cn --> Dn[... More create_view_x]
    D1 --> E[✅ success_task]
    D2 --> E
    D3 --> E
    Dn --> E
{{< /mermaid >}}
</div>

## 🚨 Lessons Learned (So You Don't Have To)

> **Pro Tip:** Always validate your sources. The GCS sensor saved me countless hours of debugging. File missing? The pipeline stops gracefully instead of failing mysteriously downstream.

- **Schema autodetection is your friend.** With `autodetect=True`, BigQuery handles schema inference automatically. No more manually defining 50+ columns!
- **Parallel processing wins.** By creating country tables in parallel, what used to take 30 minutes now completes in under 5 minutes.
- **Views > Tables for reporting.** The final reporting layer uses views instead of materialized tables. This keeps storage costs low while providing fresh data for dashboards.

## 🌍 Real-World Applications

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

## 👤 Meet the Author

Hey, I'm Vedant! I love building data systems that just work—even when the world is messy. If you're building similar pipelines or have questions about this approach, I'd love to hear from you! Data engineering is a journey best traveled with fellow practitioners who understand the joy of a well-orchestrated pipeline and the pain of debugging a 3 AM failure.

> **Let's connect:**
> - [Twitter](https://x.com/VedantJangid2)
> - [GitHub](https://github.com/vedantjangid)
> - [LinkedIn](https://www.linkedin.com/in/vedant-jangid-0a1907204/)