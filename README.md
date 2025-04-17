# NYC-Taxi-Data-Engineering-Project – Microsoft Fabric

This project demonstrates a complete end-to-end data pipeline using Microsoft Fabric to process, transform, and analyze New York City taxi trip data. It showcases key components such as the Lakehouse architecture and automated orchestration using Data Factory.

---

## Project Goals

- Ingest and process NYC taxi trip data from ADLS Gen2
- Implement Medallion architecture: Bronze, Silver, and Gold layers
- Automate data pipelines using Fabric Data Factory
- Trigger data processing on file arrival with Data Activator

---

This project is a comprehensive solution for creating an end-to-end pipeline for data processing, transformation and visualization using Microsoft Fabric, including Lakehouse, Data Factory, Dataflows Gen2, SQL Stored Procedures.

Thanks to [Mr.Malvik Vaghadia](udemy.com/course/microsoft-fabric-the-ultimate-guide) for the project inspiration.

## Technology Stack
- Microsoft Fabric (Lakehouse, Data Factory, Notebooks)
- Azure ADLS Gen2
- Power BI
- Data Activator
- SQL & Python

## Solution architecture
| |
| ----------- |
![Screenshot 2025-04-17 200501](https://github.com/user-attachments/assets/4980564a-d22a-4083-8588-a0620eb10855)

# Project Implementation Stages 

## Setting Up the Development Environment

##  Set Up ADLS Gen2 & Lakehouse

- Set Up ADLS Gen2 & Lakehouse
- Created an Azure Data Lake Storage Gen2 account.
- Added landing-zone container.
- Built a Lakehouse in Microsoft Fabric named NYC_Project_Lekahouse.
- Inside the Lakehouse, created three schemas: Bronze, Silver, and Gold for each data processing layer.
- Created a shortcut in Fabric Lakehouse to the landing-zone container.

| Container|
| ----------- |
![image](https://github.com/user-attachments/assets/d4c9015d-7059-43e2-b52d-652454c81c3c)

|Fabric Lakehouse |
| ----------- |
![image](https://github.com/user-attachments/assets/f6e48a95-8ab0-4e8f-98ff-f99e17ef8b75)

## Create Data Pipeline
Built a Data Pipeline with these three main activities:
- Set processing timestamp as a pipeline variable.
- Bronze layer processing (raw → cleaned).
- Silver layer processing (structured & enriched).
- Gold layer processing (business-ready format).

|Data Pipeline|
| ----------- |
![Screenshot 2025-04-17 191016](https://github.com/user-attachments/assets/ace78cd8-1865-40ec-b2d3-05d1d12ad741)
reate and Link Notebooks

## Created and configured three notebooks:



|Notebook 1: Bronze processing (read raw, validate, clean, archive).|
| ----------- |
```
# importing libraries
from pyspark.sql.functions import to_timestamp, col, lit

```

```
# parameters
processing_timestamp = ""
```

```
# parameters
# df now is a Spark DataFrame containing parquet data from "Files/nyc-yellow-taxi/landing_files/".
df = spark.read.format("parquet").load("Files/landing-zone/*")

# adding a processing timestamp
df = df.withColumn("processing_timestamp", to_timestamp(lit(processing_timestamp)))

# saving the data in the bronze layer table
# df.write.mode("append").saveAsTable("bronze.nyc_taxi_yellow")
df.write.mode("append").save("Tables/bronze/nyc_taxi_yellow")
```


|Notebook 2: Silver processing (apply business rules, join lookup tables).|
| ----------- |



|Notebook 3: Gold processing (aggregations, time formatting, final output).|
| ----------- |





