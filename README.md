

---

## 📌 Project Overview

This project is a complete, industry-level Azure Data Factory (ADF) 
implementation covering data ingestion from multiple sources, incremental 
loading, data transformation using Medallion Architecture (Bronze → Silver → Gold), 
alerting with Azure Logic Apps, and CI/CD integration via Azure DevOps and GitHub.

---

## 🛠️ Technologies Used

- **Azure Data Factory (ADF)** – Core orchestration and pipeline tool
- **Azure Data Lake Storage Gen2** – Cloud storage for Bronze, Silver, Gold layers
- **Azure SQL Database** – Source for relational/fact data
- **Azure Logic Apps** – Email alerting on pipeline failure/success
- **Azure DevOps** – CI/CD, Git integration, branch management, ARM templates
- **GitHub** – Code repository and version control
- **Self-Hosted Integration Runtime** – On-premises data connectivity
- **Delta Lake** – Open table format for Silver and Gold layers
- **ADF Data Flows** – Low-code data transformation using Apache Spark
- **REST API / HTTP Connector** – Data ingestion from GitHub raw URLs
- **Parquet, JSON, CSV** – Multiple file formats handled across layers

---

## 🏗️ Architecture – Medallion Pattern

- **Bronze Layer** → Raw data ingested as-is from all sources
- **Silver Layer** → Cleaned, transformed, enriched data (Delta format)
- **Gold Layer**  → Aggregated business views ready for reporting

---

## 📂 Data Sources

### 1. On-Premises Files (Local File System)
- Created Self-Hosted Integration Runtime (SHIR) and installed on local machine
- Registered SHIR using authentication key from ADF portal
- Disabled local folder path validation using PowerShell commands
- Created Linked Service: File System type with Microsoft account credentials
- Files used: dim_airline.csv, dim_flight.csv, dim_passenger.csv
- Built parameterized dataset with p_file_name parameter
- Used ForEach Activity with array of dictionaries to dynamically copy all files
- Destination: bronze/onprem/ folder in Data Lake (CSV format)

### 2. REST API / GitHub
- Used HTTP Linked Service with GitHub raw URL as Base URL
- Created JSON format dataset for dim_airport.json
- Used Copy Activity to pull data directly from GitHub
- Destination: bronze/github/dim_airport.json in Data Lake
- Also demonstrated Web Activity for pure REST API calls

### 3. Azure SQL Database – Incremental Load
- Created Azure SQL Database with fact_bookings table (1000 rows)
- Configured firewall rules for Azure services and client IP
- Used modern watermark technique with Data Lake JSON file (no watermark tables)
- Created bronze/monitor/last_load/last_load.json with initial date 1900-01-01
- Created bronze/monitor/empty_json/empty.json as watermark template
- Lookup 1 (lastload): Reads last_load.json → gets previous load date
- Lookup 2 (LatestLoad): Runs MAX(booking_date) on SQL → gets latest date
- Copy Activity: Loads only new rows using date range filter query
- Watermark Copy Activity: Overwrites last_load.json with new latest date
- Destination: bronze/sql/ folder in Data Lake (Parquet format)
- Tested: Initial load (889 rows) → Re-run (0 rows) → Incremental (35 new rows)

---

## ⚙️ Pipelines Built

- **on_prem_ingestion** – ForEach loop with parameterized Copy Activity for 3 CSV files
- **API_ingestion** – Web Activity + Copy Activity for GitHub JSON file
- **SQLtoDatalake** – 2 Lookups + Copy Activity + Watermark Copy for incremental load
- **DataTransformation** – ADF Data Flow with Bronze to Silver transformations
- **Silver Layer Pipeline** – Executes DataTransformation data flow
- **Gold Layer Pipeline** – Executes aggregation data flow for business views
- **Parent Pipeline** – Orchestrates all child pipelines using Execute Pipeline activities

---

## 🔄 Data Transformations – Silver Layer

All transformations done using ADF Data Flows (low-code, Spark-backed)

### DimAirline
- Derived Column: Converted country to UPPERCASE using upper() function
- Alter Row: Upsert logic (condition: 1==1)
- Sink: Delta Lake → silver/dim_airline/

### DimFlight
- Select: Renamed departure_time → departure_timestamp
- Select: Renamed arrival_time → arrival_timestamp
- Alter Row: Upsert logic
- Sink: Delta Lake → silver/dim_flight/

### DimPassenger
- Select: Renamed gender → gender_flag
- Derived Column: regexReplace() to replace M→Male and F→Female
- Filter: Kept only passengers with age > 25
- Derived Column: split(full_name,' ')[1] to extract first_name as new column
- Alter Row: Upsert logic
- Sink: Delta Lake → silver/dim_passenger/

### FactBookings
- Cast: Converted ticket_cost from decimal to integer
- Alter Row: Upsert with booking_id as key column
- Sink: Delta Lake → silver/fact_bookings/

### DimAirport
- Source: JSON file from bronze/github/ using inline Delta dataset
- Derived Column: Converted airport_name to lowercase using lower() function
- Alter Row: Upsert with airport_id as key
- Sink: Delta Lake → silver/dim_airport/

---

## 🥇 Gold Layer – Business Views

- Sources: Silver Delta Lake tables (DimAirline, DimFlight, FactBookings)
- Join: Left Outer Join between FactBookings and DimAirline on airline_id
- Select: Removed duplicate airline_id column after join
- Aggregate: Grouped by airline_name → SUM(ticket_cost) AS total_sales
- Window Function: dense_rank() ordered by total_sales DESC → ranking column
- Filter: Kept only top 5 airlines (ranking < 6)
- Sink: Delta Lake → gold/business_view/ (overwrite each run)

---

## 🔔 Alerting – Azure Logic Apps

- Created Azure Logic App resource in same resource group
- Designed workflow with HTTP Request trigger (POST method)
- Defined JSON body schema: pipeline_name, run_id, status, error
- Added Outlook action: Send Email (V2) with dynamic content
- Connected Web Activity in ADF to Logic App URL on pipeline failure
- Used system variables: @pipeline().Pipeline, @pipeline().RunId
- Applied if() expression for error message handling (try-catch pattern)
- Verified email delivery (check spam folder if not in inbox)

---

## ⏰ Pipeline Triggers

- Created Schedule Trigger on Parent Pipeline
- Set recurrence: Daily at 6:00 AM
- Trigger type: Schedule Trigger (most common in real-world projects)

---

## 🔧 Git Integration & Azure DevOps

### GitHub Integration
- Authorized Azure Data Factory OAuth app to access GitHub account
- Selected repository: AzureAirLineProject
- Set Collaboration Branch: main
- Initialized repository with README file before connecting (required)

### Azure DevOps Integration
- Created Azure DevOps organization and project: ADF Project
- Connected ADF to ADO via Manage → Git Configuration → Azure DevOps Git
- Set Collaboration Branch: main, Publish Branch: adf_publish
- Workflow: Feature Branch → Develop → Pull Request → Approve → Merge to main
- Clicked Publish All to generate ARM Templates in adf_publish branch

---

## 📁 Data Lake Folder Structure

bronze/
└── onprem/           → dim_airline.csv, dim_flight.csv, dim_passenger.csv
└── github/           → dim_airport.json
└── sql/              → fact_bookings (parquet files)
└── monitor/
      ├── last_load/  → last_load.json (watermark date tracker)
      └── empty_json/ → empty.json (template file)

silver/
└── dim_airline/      → Delta format
└── dim_flight/       → Delta format
└── dim_passenger/    → Delta format
└── dim_airport/      → Delta format
└── fact_bookings/    → Delta format

gold/
└── business_view/    → Delta format (Top 5 airlines by revenue)

---

## 💡 Key Concepts Learned

- Linked Services vs Datasets – Connection vs specific file/table reference
- Self-Hosted Integration Runtime – For on-prem and private network data
- Incremental Loading with JSON watermark file (modern approach)
- Dynamic pipelines using ForEach, parameters, array of dictionaries
- Delta Lake Upsert – Insert new + Update existing using key column
- Window Functions (dense_rank) in ADF Data Flows
- Logic Apps for production-ready alerting on pipeline failures
- Git branching strategy for collaborative data engineering
- ARM Template generation for environment promotion Dev → QA → Prod

---
