# Version Matrix — pinned Day 6, verified against live release pages

| Tool | Version | Date checked | Source | Compatibility Notes |
|------|---------|--------------|--------|---------------------|
| Python | 3.12.x | 05/09/2026   | python.org |                 |
| Apache Airflow | 3.1.8 | 05/09/2026 | airflow.apache.org | Airflow 2 reached EOL on April 22, 2026. |
| Apache Spark | 4.0.1 | 05/09/2026 | spark.apache.org |   |
| Apache Iceberg | 1.11.0 | 05/09/2026 | iceberg.apache.org | Spark 4.0 runtime jar: org.apache.iceberg:iceberg-spark-runtime-4.0_2.13:1.11.0 |
| pyiceberg | 0.12.0 | 05/09/2026 | pypi.org/project/pyiceberg/ |  |
| dbt-core | 1.12.0 | 05/09/2026 | docs.getdbt.com/docs/dbt-versions?version=2 | |
| dbt-bigquery | 1.12.0 | 05/09/2026 | pypi.org/project/dbt-bigquery/#history | |
| Terraform | 1.16.1 | 05/09/2026 | github.com/hashicorp/terraform/releases | |
| uv | 0.11.29 | 05/09/2026 | github.com/astral-sh/uv/releases | |
| Airflow - FAB (Flask App Builder) provider | 3.4.0 | 13/09/2026 | Bundled with apache/airflow:3.1.8 Docker image | Confirmed via `docker run --rm --entrypoint pip apache/airflow:3.1.8 list \| grep -E "providers-fab"` |
| Airflow - Google Cloud provider | 20.0.0 | 13/09/2026 | Bundled with apache/airflow:3.1.8 Docker image | Confirmed via `docker run --rm --entrypoint pip apache/airflow:3.1.8 list \| grep -E "providers-google"` |
| Terraform - Google Cloud provider | 7.39.0 | 05/09/2026 | registry.terraform.io/providers/hashicorp/google/latest | |
