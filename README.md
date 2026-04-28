# Azure End-to-End Data Engineering Pipeline
### Customer Demographics & Sales Analytics

![Architecture](images/architecture.pdf)

> A production-grade data pipeline built on Azure that ingests raw sales and customer data, transforms it through a Medallion architecture, and delivers interactive KPI dashboards to business stakeholders — refreshed daily with zero manual intervention.

---

## Business Context

A company identified a critical gap in understanding how **customer gender distribution** influences product purchasing behaviour. Key stakeholders required a comprehensive KPI dashboard that could answer:

- How is revenue distributed across male and female customers?
- Which product categories drive the most sales per demographic?
- What is the total products sold and total revenue at any given time?

This project delivers that solution through a fully automated, end-to-end Azure data pipeline.

---

## Architecture Overview

![Pipeline Architecture](images/architecture.pdf)

The solution follows the **Medallion Architecture** (Bronze → Silver → Gold), a modern lakehouse pattern that ensures data quality, traceability, and performance at each layer.

| Layer | Tool | Format | Purpose |
|---|---|---|---|
| Ingestion | Azure Data Factory | — | Extract from on-prem SQL Server |
| Bronze | ADLS Gen2 | Parquet | Raw data, unmodified |
| Silver | ADLS Gen2 | Delta | Cleaned, standardised dates |
| Gold | ADLS Gen2 | Delta | Renamed columns, report-ready |
| Serving | Azure Synapse Analytics | SQL Views | Serverless query layer |
| Visualisation | Power BI | — | KPI dashboard for stakeholders |

---

## Pipeline Walkthrough

### 1. Ingestion — Azure Data Factory
![ADF Pipeline](images/adf_pipeline.pdf)

- A **Self-Hosted Integration Runtime** connects ADF to the on-premises SQL Server (AdventureWorksLT2022)
- A **Lookup activity** dynamically fetches all table names from the source schema
- A **ForEach activity** iterates over each table and runs a **Copy activity** to land raw Parquet files in the Bronze layer of ADLS Gen2
- A **Daily Trigger** automates the entire pipeline, ensuring fresh data every 24 hours

### 2. Transformation — Azure Databricks (PySpark)

Two notebooks handle the layer-to-layer transformations:

**Bronze → Silver**
- Standardises date columns to a consistent `yyyy-MM-dd` format
- Writes output as Delta format for ACID compliance

**Silver → Gold**
- Renames columns from `PascalCase` to `snake_case` for SQL and BI tool compatibility
- Produces the final, analytics-ready Delta tables

### 3. Serving — Azure Synapse Analytics (Serverless SQL Pool)

- A stored procedure dynamically creates **SQL views** on top of each Gold Delta table using `OPENROWSET`
- A Synapse pipeline with a **ForEach activity** calls the stored procedure for each table
- Views are exposed via the serverless `-ondemand` SQL endpoint, requiring no dedicated compute

### 4. Visualisation — Power BI

![Power BI Dashboard](images/powerbi_dashboard.pdf)

The dashboard connects directly to Synapse Serverless and surfaces the following KPIs:

- **Number of Products** — distinct count of products sold
- **Total Sales Revenue** — sum of all line totals
- **Gender Split** — donut chart showing male vs. female customer distribution
- **Filters** — slicers for Product Category and Customer Title (gender proxy)

---

## Tech Stack

| Category | Technology |
|---|---|
| Orchestration | Azure Data Factory |
| Compute | Azure Databricks (PySpark) |
| Storage | Azure Data Lake Storage Gen2 |
| Serving | Azure Synapse Analytics (Serverless SQL Pool) |
| Visualisation | Power BI Service |
| Security | Azure Key Vault, Azure Entra ID (RBAC) |
| Source System | SQL Server — AdventureWorksLT2022 |
| Version Control | GitHub (this repo) |

---

## Security & Governance

- **Azure Key Vault** stores all sensitive credentials including storage account keys and SQL passwords — no plain-text secrets exist anywhere in the pipeline code
- **Managed Identity** is used for service-to-service authentication between Synapse and ADLS Gen2, eliminating the need for stored credentials
- **Role-Based Access Control (RBAC)** via Azure Entra ID ensures only authorised identities can access each resource
- **Storage Blob Data Reader/Contributor** roles are scoped to the Synapse workspace managed identity

---

## Repository Structure

```
azure-customer-sales-analytics-pipeline/
├── images/                         # Architecture and dashboard screenshots
├── databricks_notebook/
│   ├── bronze to silver.ipynb      # Bronze → Silver transformation
│   ├── silver to gold.ipynb        # Silver → Gold transformation
│   └── storagemount.ipynb          # ADLS mount configuration
├── pipeline/                       # ADF pipeline definitions (JSON)
├── dataset/                        # ADF dataset definitions
├── linkedService/                  # ADF linked service configs
├── trigger/                        # ADF trigger definitions
├── sqlscript/                      # Synapse SQL scripts
├── intech-synapse30/               # Synapse workspace resources
└── powerbi/
    └── customer report.pbix        # Power BI report file
```

---

## How to Reproduce

1. **Provision Azure resources**: ADLS Gen2, Azure Databricks, Azure Synapse Analytics, Azure Data Factory, Azure Key Vault
2. **Configure Key Vault**: Store storage account key and SQL credentials as secrets
3. **Set up ADLS containers**: Create `bronze`, `silver`, `gold` containers
4. **Deploy ADF pipeline**: Import JSON definitions from the `pipeline/` folder
5. **Run Databricks notebooks**: Execute `storagemount.ipynb` first, then run the transformation notebooks
6. **Deploy Synapse stored procedure**: Run the SQL scripts from `sqlscript/` folder in `gold_db`
7. **Connect Power BI**: Connect to the Synapse serverless endpoint and import the `.pbix` file

---

## Key Learnings

- Implemented dynamic pipeline patterns (Lookup + ForEach) to avoid hardcoding table names
- Configured Managed Identity authentication between Azure services to eliminate credential exposure
- Resolved Delta Lake compatibility issues between Databricks writer versions and Synapse Serverless reader
- Used serverless SQL pools as a cost-efficient serving layer — no dedicated compute required

---

*Built as part of an end-to-end Azure Data Engineering portfolio project.*
