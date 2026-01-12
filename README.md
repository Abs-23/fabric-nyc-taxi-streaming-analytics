# NYC Taxi Fleet Analytics – Microsoft Fabric

A batch and streaming analytics pipeline built on Microsoft Fabric exploring mobility data processing patterns.

---

## Project Overview

This project explores end-to-end data engineering workflows using NYC taxi trip data. It combines batch historical analysis with real-time streaming to replicate the data architecture mobility companies use for fleet operations.

**Key technical features:**
- Dual pipelines (batch and streaming) that merge into a unified gold layer
- KQL testing layer for validating streaming transformations before production deployment
- Incremental watermarking to process only new streaming data
- Weather data integration to analyze demand patterns under different conditions

The architecture demonstrates medallion design (bronze/silver/gold), scheduled automation, and Power BI semantic modeling.

---

## Business Context

The system addresses common operational questions in fleet management:

- **Fleet positioning**: Which zones need more cars? Where are cars sitting idle?
- **Revenue analysis**: What are the revenue trends? How does payment mix vary by zone?
- **Demand forecasting**: When are peak hours? Are staffing levels appropriate?

The dashboards and data model support these decision patterns.

---

## Dashboard Pages

Five Power BI pages explore different aspects of taxi operations.

### 1. Revenue Analysis
Shows total revenue, trip counts, and average fares by zone with filters for date range and time of day. The scatter plot compares trip volume against revenue to identify high-performing zones.

<img width="1740" height="802" alt="Image" src="https://github.com/user-attachments/assets/449b3cdd-482c-41bd-9fba-01b3ba6eca7c" />

---

### 2. Demand Patterns
Displays hourly and daily trip patterns with weekday vs weekend comparisons. The heat map shows trip volume by date and hour, while the trend line tracks how demand changes across the month.

<img width="1737" height="802" alt="Image" src="https://github.com/user-attachments/assets/2b574aef-136e-4717-af18-eb2537128937" />

---

### 3. Trip Duration & Distance
Analyzes which zones have the longest average trips and how distance correlates with duration throughout the day. The scatter plot reveals zones where short trips take unusually long, indicating potential traffic bottlenecks.

<img width="1737" height="806" alt="Image" src="https://github.com/user-attachments/assets/d356dcad-c4e0-4b76-bf8b-ea3cc613eed2" />

---

### 4. Weather Impact
Combines trip data with weather conditions to show how rain and snow affect demand and average fares. The breakdown by zone reveals which areas see the biggest surge during adverse weather.

<img width="1738" height="802" alt="Image" src="https://github.com/user-attachments/assets/2d1541f8-5562-4301-bc15-2419660236a0" />

---

### 5. Live Status (Streaming)
Displays real-time trip volume and revenue by hour, updated every 15 minutes through the Fabric Eventstream pipeline. The combo chart shows both demand and revenue patterns as they happen.

<img width="1738" height="802" alt="Image" src="https://github.com/user-attachments/assets/03097570-6248-46e3-a784-220ae58541c6" />

---

## How It Works

### Batch Pipeline (Historical Analysis)

**Data sources:**
- NYC Yellow Taxi trip data (Parquet file with 2.8M records)
- Weather data (CSV file with daily conditions)

**Processing flow:**

1. **Bronze layer**: Fabric pipelines ingest raw Parquet and CSV files into Lakehouse tables
2. **Silver layer**: PySpark notebooks clean the data by removing nulls, deduplicating trips, and joining weather data by date
3. **Gold layer**: SQL queries aggregate data into business metrics (trips by zone/hour, revenue by payment type, average fares)
4. **Semantic model**: A dimensional model with date, time, payment type, and zone dimensions feeds fact tables for Power BI

<img width="1912" height="818" alt="Image" src="https://github.com/user-attachments/assets/68a04edf-3037-4ac4-95ff-e28de8e186b0" />

---

### Streaming Pipeline (Real-Time Monitoring)

**Data source:**
- Fabric Eventstream with Yellow Taxi sample data (simulates live trips)

**Processing flow:**

1. **Ingestion**: Eventstream captures live taxi events and routes them to two destinations:
   - Eventhouse (KQL database) for fast testing and validation
   - Lakehouse streaming table for integration with the batch gold layer

2. **KQL validation**: All transformations (aggregations, filters, timestamp parsing) are tested in KQL before applying to production tables. This sandbox approach prevents bad data from corrupting gold tables.

3. **Gold conversion with incremental watermarking**: A scheduled notebook runs every 15 minutes with custom incremental processing logic:
   - Reads the maximum `pickup_ts` (hour timestamp) from the gold table as a watermark
   - Filters silver streaming data to only process rows with `pickup_ts` greater than the watermark
   - Aggregates new data by pickup date, hour, and zone
   - Calculates trip counts, revenue totals, average fares, trip distances, durations, and tip percentages
   - Inserts only new aggregated rows into gold (no duplicates, no re-processing)
   - On the first run (empty gold table), processes all available streaming data
   - Logs pipeline run status and performs data quality checks on the processed window

4. **Power BI integration**: The gold streaming table connects to the existing semantic model for near-real-time dashboard updates.

<img width="1919" height="815" alt="Image" src="https://github.com/user-attachments/assets/0c432358-2288-44ed-b767-e235368608ca" />
*Eventstream ingestion in action*

<img width="1918" height="827" alt="Image" src="https://github.com/user-attachments/assets/6dbc4113-594b-4ed4-a7ac-c4abbd8615ca" />
*KQL database metrics showing streaming throughput*

---

## Key Features

**Medallion architecture**: Organizes data from raw ingestion (bronze) through cleaning (silver) to business metrics (gold)

**KQL testing environment**: Validates streaming transformations in Eventhouse before deploying to production, preventing corrupted gold tables

**Incremental streaming processing**: Custom watermarking logic tracks the latest processed hour in the gold table and filters silver data to only process new records, preventing duplicates and reducing compute on each run

**Scheduled automation**: Runs a notebook every 15 minutes with pipeline logging (run IDs, status, rows written) and data quality checks (trip counts, average amounts, percentage of invalid trip distances in the processed window)

**Weather enrichment**: Joins external weather data to taxi trips to explore how rain, snow, or clear conditions affect demand patterns

**Streaming and batch alignment**: Both pipelines land in the same gold schema with validation queries confirming minimal variance over overlapping time windows

**Row-level security**: Power BI model includes RLS rules so regional managers see only their zones while executives see all data

**Direct Lake mode**: Power BI reads Delta tables in Fabric without importing or exporting data, enabling faster refreshes and lower latency

---

## Data Quality & Monitoring

The scheduled notebook includes two validation mechanisms:

**Pipeline run logs:**
- Tracks each execution with unique run IDs and timestamps
- Records source and target table names
- Counts rows written to gold tables
- Logs success or failure status with descriptive messages

**Streaming data quality checks:**
- Validates the processed data window on each run
- Monitors total trip volume to detect ingestion issues
- Tracks average transaction amounts to catch anomalies
- Calculates percentage of trips with invalid distances (≤0) to identify bad data

These checks write to dedicated tables (`pipeline_run_log` and `dq_streaming_checks`) for historical monitoring and troubleshooting.

---

## Tech Stack

**Platform**: Microsoft Fabric (Lakehouse, Eventstream, Eventhouse, Data Pipelines)  
**Processing**: PySpark (batch transformations, watermark logic), SQL (aggregations), KQL (streaming validation)  
**Storage**: Delta Lake (bronze/silver/gold tables)  
**Analytics**: Power BI with DAX measures (total revenue, avg distance, trip counts), Direct Lake mode  
**Orchestration**: Scheduled notebooks (15-minute intervals), Fabric pipelines for batch ingestion  
**Validation**: Automated logging (row counts, null checks, timestamp alignment)

---

## Semantic Model Structure

**Dimensions:**
- `dim_date`: calendar dates, weekdays, month/year hierarchy
- `dim_time`: hourly granularity for peak analysis
- `dim_zone`: NYC taxi zones (Manhattan, Brooklyn, etc.)

**Fact tables:**
- `gold_zone_hour_metrics`: hour-level data (distance, fare, tip, passenger count)
- `gold_zone_day_metrics`: daily-level data
- `gold_zone_hour_metrics_streaming`: hourly aggregates from live data

Tables are connected via star schema for fast Power BI queries.

<img width="1840" height="736" alt="Image" src="https://github.com/user-attachments/assets/41d0b46b-1382-426e-bd6a-0a2efcae1659" />

---

## How to Run This

To run this project:

1. **Fabric trial**: Sign up at fabric.microsoft.com and create a workspace and lakehouse
2. **Upload data**: Add the sample Parquet and CSV files to Lakehouse Files from the data folder
3. **Run notebooks sequentially**:
   - `01_medallion_ingestion_silver.ipynb` to ingest and clean raw data
   - `02_dimension_modeling_gold.ipynb` to build dimensions and fact tables
   - `03_streaming_gold_pipeline_logs.ipynb` to simulate the streaming gold layer (run on schedule)
4. **Set up Eventstream**: Point to Yellow Taxi sample or your own feed and route to Eventhouse and Lakehouse
5. **Import Power BI report**: Open `NYC_Taxi_Analytics.pbix` and connect to your Fabric semantic model

Notebooks run standalone in VS Code or Databricks Community if you hit Fabric trial limits.

---

## Skills Covered

- Batch and streaming data processing in Microsoft Fabric
- Incremental watermarking for efficient stream processing
- Medallion architecture (bronze/silver/gold layers)
- Power BI semantic modeling with star schema
- Data quality validation and pipeline logging
- KQL for real-time data validation

---
