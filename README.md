# NYC-Taxi-Data-Engineering-Project – Microsoft Fabric

This project demonstrates a complete end-to-end data pipeline using Microsoft Fabric to process, transform, and analyze New York City taxi trip data. It showcases key components such as the Lakehouse architecture and automated orchestration using Data Factory.

---

## Project Goals

- Ingest and process NYC taxi trip data from ADLS Gen2
- Implement Medallion architecture: Bronze, Silver, and Gold layers
- Automate data pipelines using Fabric Data Factory
- Trigger data processing on file arrival with Data Activator

---

This project is a comprehensive solution for creating an end-to-end pipeline for data processing, transformation using Microsoft Fabric, including Lakehouse, Data Factory, Dataflows Gen2, SQL Stored Procedures.

Thanks to [Mr.Malvik Vaghadia](https://www.udemy.com/course/microsoft-fabric-the-ultimate-guide) for the project inspiration.

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

# Setting Up the Development Environment

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
- Delete landing zone to manage and clean the storage.
- Silver layer processing (structured & enriched).
- Gold layer processing (business-ready format).

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
# df now is a Spark DataFrame containing parquet data from "Files/nyc-yellow-taxi/landing_files/".
df = spark.read.format("parquet").load("Files/landing-zone/*")

# adding a processing timestamp
df = df.withColumn("processing_timestamp", to_timestamp(lit(processing_timestamp)))

# saving the data in the bronze layer table
df.write.mode("append").save("Tables/bronze/nyc_taxi_yellow")
```


|Notebook 2: Silver processing (apply business rules, join lookup tables).|
| ----------- |

```
# importing libraries
from pyspark.sql.functions import to_timestamp, col, current_timestamp, expr
```

```
# parameters
processing_timestamp = ""
```

````

# df now is a Spark DataFrame containing parquet data from the bronze table
df = spark.read.format("delta").load("Tables/bronze/nyc_taxi_yellow")

# filtering to only process latest batch of data
df = df.filter(f"processing_timestamp = '{processing_timestamp}'")


# sql case statements
vendor_case_sql = """ 
case 
    when VendorID = 1 then 'Creative Mobile Technologies'
    when VendorID = 2 then 'VeriFone'
    else 'Unknown'
end
"""

payment_method_sql = """
case 
    when payment_type = 1 then 'Credit Card'
    when payment_type = 2 then 'Cash'
    when payment_type = 3 then 'No Charge'
    when payment_type = 4 then 'Dispute'
    when payment_type = 5 then 'Unknown'
    when payment_type = 6 then 'Voided Trip'
    else 'Unknown'
end
"""

# using sql case statements to add vendor and payment_method columns
# selecting columns
df = df.\
        withColumn("vendor", expr(vendor_case_sql)).\
        withColumn("payment_method", expr(payment_method_sql)).\
        select(
                "vendor",
                "tpep_pickup_datetime",
                "tpep_dropoff_datetime",
                "passenger_count",
                "trip_distance",
                col("RatecodeID").alias("ratecode_id"),
                "store_and_fwd_flag",
                col("PULocationID").alias("pu_location_id"),
                col("DOLocationID").alias("do_location_id"),
                "payment_method",
                "fare_amount",
                "extra",
                "mta_tax",
                "tip_amount",
                "tolls_amount",
                "improvement_surcharge",
                "total_amount",
                "congestion_surcharge",
                col("Airport_fee").alias("airport_fee"),
                "processing_timestamp"
                )

# saving the data in the bronze layer table
df.write.mode("append").save("Tables/silver/nyc_taxi_yellow")

````


|Notebook 3: Gold processing (aggregations, time formatting, final output).|
| ----------- |

```
# importing libraries
from pyspark.sql.functions import to_timestamp, col, current_timestamp, expr, date_format
```


```
# parameters
processing_timestamp = ""
```


```
# reading the nyc_taxi_yellow data (for the latest processed batch) into a dataframe df
df = spark.read.format("delta").load("Tables/silver/nyc_taxi_yellow").filter(f"processing_timestamp = '{processing_timestamp}'")

# reading the lookup data into a dataframe df_pu_lookup
df_pu_lookup = spark.read.format("delta").load("Tables/silver/taxi_zone_lookup")

# reading the lookup data into a dataframe df_do_lookup
df_do_lookup = spark.read.format("delta").load("Tables/silver/taxi_zone_lookup")

```

```
# joining df with df_po_lookup to get pickup information
df = df.join(df_pu_lookup, df["pu_location_id"]==df_pu_lookup["LocationID"], "left")

# joining df with df_do_lookup to get dropoff information        
df = df.join(df_do_lookup, df["do_location_id"]==df_do_lookup["LocationID"], "left")


# selecting only required columns from df
# performing transformations
df = df.\
        select(
                "vendor", 
                # changing to standard date format and aliasing columns
                date_format("tpep_pickup_datetime","yyyy-MM-dd").alias("pickup_date"), 
                date_format("tpep_dropoff_datetime","yyyy-MM-dd").alias("dropoff_date"), 
                df_pu_lookup["Borough"].alias("pickup_borough"), 
                df_do_lookup["Borough"].alias("dropoff_borough"), 
                df_pu_lookup["Zone"].alias("pickup_zone"), 
                df_do_lookup["Zone"].alias("dropoff_zone"), 
                "payment_method", 
                "passenger_count", 
                "trip_distance", 
                "tip_amount", 
                "total_amount",
                "processing_timestamp" )

# writing to gold table
df.write.mode("append").save("Tables/gold/nyc_taxi_yellow")

```

|Data Pipeline|
| ----------- |
![Screenshot 2025-04-17 191016](https://github.com/user-attachments/assets/ace78cd8-1865-40ec-b2d3-05d1d12ad741)

|Set processing_timestamp|
| ----------- |
![image](https://github.com/user-attachments/assets/a58c0708-cef4-48ee-817d-a9f1910db3f7)

|Bronze Layer Processing|
| ----------- |
![image](https://github.com/user-attachments/assets/9ded7ceb-f06d-4764-9a9c-f49a69bf8af2)

|Delete Landing Zone Data|
| ----------- |
![image](https://github.com/user-attachments/assets/04c4c3f2-84d0-4a25-bea2-4a7c9f25cebf)

|Silver Layer Processing|
| ----------- |
![image](https://github.com/user-attachments/assets/60454ca4-0e29-46da-9502-667a24291c1b)

|Gold Layer Processing|
| ----------- |
![image](https://github.com/user-attachments/assets/e613d136-b05e-448e-9190-1eaa53afdace)



## Gold Table After Pipeline Run – January
| |
| -----------  |
![image](https://github.com/user-attachments/assets/02a04ec6-a24a-4299-b1ba-00d7f1ef248c)

## Timestamp Results with Total Records – Table After Pipeline Run – January
| |
| -----------  |
![image](https://github.com/user-attachments/assets/47df7af0-732a-4875-8289-673caaddcc93)

## February Data Added & Pipeline Re-run
| |
| -----------  |
![image](https://github.com/user-attachments/assets/aeea1479-dea4-4461-b256-30445aeeadcf)
![image](https://github.com/user-attachments/assets/ef9129dc-413c-4647-a42e-6bc68df51b8b)
![image](https://github.com/user-attachments/assets/3a279e26-4a97-4ebc-bfa7-a4ded4d7d819)


## Gold Table After Pipeline Run – February
| |
| -----------  |
![image](https://github.com/user-attachments/assets/ccf45f14-e514-453f-8ef6-665bb3089832)


## Timestamp Results with Total Records – Table After Pipeline Run – February
| |
| -----------  |
![image](https://github.com/user-attachments/assets/02f30cfb-4b26-42ad-bcd9-b9acf1f6b43f)


# Next Step: Creating a Trigger for Automated Pipeline Runs

| Creating a trigger is to connect to the storage account where your data files are being uploaded|
| -----------  |
![image](https://github.com/user-attachments/assets/6e6a8b1e-fe19-465e-9dc7-58be7c0f286f)
![image](https://github.com/user-attachments/assets/db1f6242-48c2-4fe9-9f98-941c7528a60f)
![image](https://github.com/user-attachments/assets/2a3d6327-4e64-4a93-aa0f-3303d65db4ad)
![image](https://github.com/user-attachments/assets/1b50d947-294b-42de-a602-96a0b3438d44)

## March Data Added & Trigger Pipeline Re-run
![image](https://github.com/user-attachments/assets/cc4ea136-a822-4885-9c87-b924a1c9a2e0)
![image](https://github.com/user-attachments/assets/e549f3eb-1c0a-4231-a92d-a18f4700a068)

## Gold Table After Trigger Run Pipeline – March
| |
| -----------  |
![image](https://github.com/user-attachments/assets/6325c4f8-d4bd-424f-937f-b63b653035e3)

## Timestamp Results with Total Records – Table After Trigger Run Pipeline – March
| |
| -----------  |
![image](https://github.com/user-attachments/assets/b0239896-d8f8-402c-84d5-a64abbe299d4)



# Conclusion 
In this project, I successfully designed and implemented an end-to-end data pipeline for processing NYC Taxi trip data. The pipeline ingests raw data, performs necessary transformations, and appends the processed data to a Gold table. I also implemented an event-based trigger to automate the execution of the pipeline whenever new data is uploaded to the storage container.

Key Achievements:

Data Integration & Transformation:

- Successfully processed data for January, February, and March.

- Implemented transformations to standardize and clean the data for further analysis.

Automation & Efficiency:

- Designed and documented an event-based trigger to automate pipeline execution when new data is uploaded.

- Due to permission restrictions, I manually triggered the pipeline, simulating the automation process and ensuring the pipeline runs correctly with new data.

Error Handling & Testing:

- Addressed challenges with data delivery and ensured proper error handling within the pipeline.

- Manually validated the functionality of the pipeline and trigger logic through testing and debugging.

Documentation & Reporting:

- Thoroughly documented each step of the process, from data upload to transformation and trigger setup.

- Provided clear visual evidence and screenshots to demonstrate successful pipeline execution.

Technical Skills Demonstrated:
- Microsoft Fabric (Lakehouse, Data Factory, Notebooks)
- Azure ADLS Gen2
- Power BI
- Data Activator
- SQL & Python


This project has provided me with hands-on experience in data engineering and automation. It demonstrates my ability to design scalable and efficient data workflows, integrate new data into the pipeline, and automate processing with event-based triggers. Once the necessary permissions for trigger execution are in place, the pipeline can be fully automated, enhancing efficiency and reducing manual intervention.






