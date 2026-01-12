# NYC Taxi Fleet Analytics – Microsoft Fabric

A batch and streaming analytics pipeline built on Microsoft Fabric to demonstrate data engineering skills in mobility analytics.

---

## About This Project

I built this portfolio project to show I can design data systems that solve real business problems. While I'm not working with an actual taxi company, I've replicated the workflows fleet managers, finance teams, and operations analysts use to position cars based on demand, track revenue trends, and respond to live conditions.

**What sets this apart:**
- **Dual pipelines**: Separate batch and streaming paths that merge into a unified gold layer
- **KQL testing layer**: Streaming transformations are validated in a KQL database before updating production tables
- **Incremental watermarking**: Custom logic tracks the latest processed hour to avoid duplicate data and reduce compute
- **Weather integration**: Combined taxi trips with weather data to see how conditions affect demand

This replicates the architecture a mobility company would use for fleet operations.

---

## Business Context

If a taxi operator used this system, they could answer:

- **Fleet managers**: "Which zones need more cars in the next hour? Where are cars sitting idle?"
- **Finance teams**: "What's our revenue trend this week? How does payment mix vary by zone?"
- **Operations**: "When are peak hours? Are we staffed correctly for Friday evenings?"

The dashboards and data model support these decisions rather than just displaying numbers.

---

## Dashboard Pages

I built five Power BI pages that answer specific business questions for fleet managers, finance teams, and operations analysts.

### 1. Executive Overview (Revenue Analysis)
**Business question:** "Which zones generate the most revenue, and where should we focus our fleet resources?"

This page displays three key performance indicators: total revenue ($77.94M), total trips (2.82M), and average amount per trip ($27.68). The page includes filters for service date range and time of day (morning, afternoon, evening, night) to analyze performance across different operating periods.

The top zones by total revenue chart reveals that Zone_132 and Zone_138 dominate revenue generation, while the trips vs revenue scatter plot identifies high-volume zones that may be underpriced. The average amount per trip by zone chart helps finance teams spot premium zones where customers pay significantly more per ride, informing dynamic pricing strategies.

**Business impact:** Fleet managers can allocate more cars to high-revenue zones during peak periods, while finance teams can identify zones where pricing adjustments could capture additional revenue.

<img width="1200" alt="Executive Overview" src="dashboard/01_dashboard.png)" />

---

### 2. Trends Over Time (Demand Patterns)
**Business question:** "When do we see peak demand, and how does it differ between weekdays and weekends?"

This page analyzes temporal demand patterns through multiple views. The hourly trip demand heat map shows trip volume by date and hour, revealing that hours 8-14 consistently generate 4,000-6,000+ trips. The weekday vs weekend trend line clearly shows weekday demand (blue line) remains stable around 100K trips, while weekend demand (orange line) drops sharply in late January.

The top zones by trips chart identifies Zone_237, Zone_161, and Zone_236 as the highest-volume pickup locations across the entire date range. Combined with the time-of-day filter, operations teams can identify exactly when and where to position cars for maximum utilization.

**Business impact:** Operations can optimize shift schedules and car positioning by deploying more drivers to high-demand zones during hours 8-14 on weekdays, and reducing fleet size on weekends when demand drops.

![Trends Over Time](powerbi/screenshots/lakehouse-preview.png)

---

### 3. Operational Insights (Trip Duration & Distance)
**Business question:** "Which zones have the longest trips, and how do distance and duration correlate across different hours?"

This page helps operations teams understand trip efficiency and identify problem areas. The zones with highest average duration chart shows Zone_218, Zone_86, and Zone_117 have trips averaging 60-80 minutes, suggesting these zones face congestion or serve longer-distance routes.

The distance vs duration scatter plot (with bubble size representing trip volume) reveals most zones cluster around 5 miles and 20 minutes, but several outliers show zones with unusually long durations for short distances, indicating traffic bottlenecks. The hourly duration vs distance chart shows trip duration peaks during hours 5-10 (morning rush) while distance remains relatively flat, confirming congestion patterns.

**Business impact:** Operations can adjust pricing for high-duration zones to compensate drivers for time spent in traffic, and fleet managers can avoid positioning cars in congested zones during rush hours unless premium fares justify the wait time.

![Operational Insights](powerbi/screenshots/kql-database.png)

---

### 4. Live Status Dashboard (Real-Time Monitoring)
**Business question:** "What's happening right now, and where do we need to deploy cars in the next hour?"

This streaming dashboard displays real-time metrics powered by the Fabric Eventstream pipeline. Three KPI cards show cumulative streaming data: 3.43M total trips processed, $76.23M total revenue, and $22.20 average ticket size. These update as new data flows through the system every 15 minutes.

The live demand by hour chart shows current-day trip volume, revealing a clear peak around hours 18-20 (evening rush) with 350K-400K trips. The live revenue vs trips by hour combo chart displays both trip volume (bars) and revenue (line) on the same axis, showing that the first few hours of the day (18:00-20:00 timestamp range) generate the highest revenue despite moderate trip counts, suggesting premium pricing or longer trips during those windows.

**Business impact:** Dispatchers can monitor real-time demand patterns and redirect available cars to zones experiencing surges within minutes, rather than waiting for end-of-day reports. The streaming data enables proactive fleet positioning instead of reactive adjustments.

![Live Status](powerbi/screenshots/realtime-stream.png)

---

### 5. Weather Integration (Demand by Conditions)
**Business question:** "How do weather conditions affect demand, and should we adjust fleet size during rain or snow?"

This page integrates external weather data with trip records to analyze demand sensitivity. KPI cards show that out of 2.82M total trips, 500.62K occurred during rain and 253.68K during snow, with average tickets increasing from $29.07 (rain) to $30.77 (snow), indicating customers are willing to pay more during adverse conditions.

The daily trips by weather condition chart stacks clear (light blue), rain (purple), and snow (orange) trips over time, showing clear weather dominates most days with occasional rain spikes around January 12th and 20th. The top zones by rain and snow trips chart reveals Zone_132, Zone_237, and Zone_161 see the most weather-impacted demand, suggesting these zones have customers who rely heavily on taxis when conditions worsen.

**Business impact:** Fleet managers can pre-position additional cars in high-demand zones before forecasted rain or snow events, capturing surge demand and higher average fares. Finance teams can implement dynamic weather-based pricing knowing customers accept premium rates during bad weather.

![Weather Integration](powerbi/screenshots/realtime-stream.png)

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
4. **Semantic model**: I created a dimensional model with date, time, payment type, and zone dimensions that feed fact tables for Power BI

![Lakehouse Preview](powerbi/screenshots/lakehouse-preview.png)

---

### Streaming Pipeline (Real-Time Monitoring)

**Data source:**
- Fabric Eventstream with Yellow Taxi sample data (simulates live trips)

**Processing flow:**

1. **Ingestion**: Eventstream captures live taxi events and routes them to two destinations:
   - Eventhouse (KQL database) for fast testing and validation
   - Lakehouse streaming table for integration with the batch gold layer

2. **KQL validation**: I test all transformations (aggregations, filters, timestamp parsing) in KQL before applying them to production tables. This sandbox approach prevents bad data from corrupting gold tables.

3. **Gold conversion with incremental watermarking**: A scheduled notebook runs every 15 minutes with custom incremental processing logic:
   - Reads the maximum `pickup_ts` (hour timestamp) from the gold table as a watermark
   - Filters silver streaming data to only process rows with `pickup_ts` greater than the watermark
   - Aggregates new data by pickup date, hour, and zone
   - Calculates trip counts, revenue totals, average fares, trip distances, durations, and tip percentages
   - Inserts only the new aggregated rows into gold (no duplicates, no re-processing)
   - On the first run (empty gold table), processes all available streaming data
   - Logs pipeline run status and performs data quality checks on the processed window

4. **Power BI integration**: The gold streaming table connects to the existing semantic model. The Live Status dashboard queries this table for near-real-time KPIs.

![Real-Time Stream](powerbi/screenshots/realtime-stream.png)
*Eventstream ingestion in action*

![Eventhouse Stats](powerbi/screenshots/eventhouse-stats.png)
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
- `dim_payment_type`: card, cash, mobile app
- `dim_zone`: NYC taxi zones (Manhattan, Brooklyn, etc.)

**Fact tables:**
- `fact_trips`: trip-level data (distance, fare, tip, passenger count)
- `fact_streaming_trips`: hourly aggregates from live data

I connected these tables via star schema for fast Power BI queries.

![Semantic Model](powerbi/screenshots/semantic-model.png)

---

## Who Should Use This

**Data engineers learning Fabric or preparing for DP-700:**  
Clone this repo to see lakehouse design with medallion layers, streaming integration, semantic modeling, and Power BI reports. Adjust the data source to create your own portfolio piece.

**Teams prototyping mobility, logistics, or IoT analytics:**  
Replace NYC Taxi data with delivery fleets, ride-share logs, or sensor streams. The architecture (batch plus streaming, KQL validation, gold aggregates) transfers directly.

**Hiring managers and interviewers:**  
This project shows I can:
- Design pipelines that support business decisions rather than just storing data
- Build batch and streaming paths that align and validate against each other
- Implement incremental processing patterns to handle streaming data efficiently
- Translate technical work into language for executives, finance, and operations teams
- Handle data quality challenges including nulls, duplicates, and timestamp drift

---

## How to Run This

You can reproduce this project in about 30 minutes:

1. **Fabric trial**: Sign up at fabric.microsoft.com and create a workspace and lakehouse
2. **Upload data**: Add sample Parquet and CSV files to Lakehouse Files
3. **Run notebooks sequentially**:
   - `01_medallion_ingestion_silver.ipynb` to ingest and clean raw data
   - `02_dimension_modeling_gold.ipynb` to build dimensions and fact tables
   - `03_streaming_gold_pipeline_logs.ipynb` to simulate the streaming gold layer (run on schedule)
4. **Set up Eventstream**: Point to Yellow Taxi sample or your own feed and route to Eventhouse and Lakehouse
5. **Import Power BI report**: Open `NYC_Taxi_Analytics.pbix` and connect to your Fabric semantic model

Notebooks run standalone in VS Code or Databricks Community if you hit Fabric trial limits.

---

## What This Demonstrates

I can handle ingestion, transformation, modeling, and visualization across batch and streaming pipelines. The project shows I understand how to integrate real-time and historical data into a unified gold layer while maintaining logging, validation, scheduled automation, RLS, and semantic modeling. Every technical choice supports a specific decision (zone placement, revenue tracking, demand forecasting) rather than existing for its own sake.

---

**Built by [Your Name]** – Data Engineer specializing in cloud analytics platforms and real-time data systems  
[LinkedIn](#) | [Portfolio](#)
