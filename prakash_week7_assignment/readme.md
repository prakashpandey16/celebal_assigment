# 🧾 ETL Project: Load Files from Data Lake to SQL

---

## ✅ Objective

I have 3 types of CSV files stored in a **Data Lake folder**, and my task is to:

- 🔄 Load each into their respective **SQL database tables**
- 🧹 Perform a **truncate-and-load** operation **daily**
- 📅 Extract the **date from filenames** and include it as a column for certain files

---

## 🗂️ File Types and Rules

| File Example                        | Target Table        | Transformation Required                                               |
|------------------------------------|---------------------|------------------------------------------------------------------------|
| `CUST_MSTR_20191112.csv`           | `CUST_MSTR`         | ➕ Add `date` column → `2019-11-12`                                     |
| `master_child_export-20191112.csv` | `master_child`      | ➕ Add `date` → `2019-11-12`<br>➕ Add `date_key` → `20191112`           |
| `H_ECOM_ORDER.csv`                 | `H_ECOM_Orders`     | ✅ Load as-is (no transformation)                                      |

---

# 🧰 ETL Pipeline in 5 Steps

---

### ✅ Step 1: Spark & Environment Setup

Before I start processing files, I:

- 🚀 Initialize a **Spark session**
- 📁 Set the **data lake path**
- 🔌 Configure **JDBC settings** to connect with the SQL database

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import lit
import re

# Start Spark session
spark = SparkSession.builder.appName("Daily_ETL_Pipeline").getOrCreate()

# Data lake path
data_lake_path = "/mnt/datalake/container/"  # Replace with actual path

# JDBC configuration
jdbc_url = "jdbc:sqlserver://<server>:<port>;databaseName=<dbname>"
jdbc_props = {
    "user": "<username>",
    "password": "<password>",
    "driver": "com.microsoft.sqlserver.jdbc.SQLServerDriver"
}



## 📍 Step 2: List and Loop Over Files

In this step, I:

- 🔍 List all files in the **data lake container**
- 📄 Loop through each file and apply logic based on the filename pattern

```python
files = dbutils.fs.ls(data_lake_path)

for file in files:
    filename = file.name
    file_path = file.path


### ✅ Step 3: Handle `CUST_MSTR_YYYYMMDD.csv`

In this step:

- 🧠 Detect files that start with `CUST_MSTR_`
- 🗓️ Extract the date from the filename and add it as a new column
- 🧹 Truncate and insert into the `CUST_MSTR` table

```python
    if filename.startswith("CUST_MSTR_") and filename.endswith(".csv"):
        match = re.search(r"CUST_MSTR_(\d{8})\.csv", filename)
        if match:
            date_raw = match.group(1)
            date_fmt = f"{date_raw[:4]}-{date_raw[4:6]}-{date_raw[6:]}"
            
            df = spark.read.option("header", "true").csv(file_path)
            df = df.withColumn("date", lit(date_fmt))

            # Truncate and load
            spark.sql("TRUNCATE TABLE CUST_MSTR")
            df.write.jdbc(url=jdbc_url, table="CUST_MSTR", mode="append", properties=jdbc_props)

            print(f"✅ Loaded: {filename} into CUST_MSTR")



### ✅ Step 4: Handle `master_child_export-YYYYMMDD.csv`

In this step:

- 🧠 I detect files that start with `master_child_export-`
- 🗓️ Extract both `date` and `date_key` from the filename
- ➕ Add them as columns
- 🧹 Truncate and insert into the `master_child` table

```python
    elif filename.startswith("master_child_export-") and filename.endswith(".csv"):
        match = re.search(r"master_child_export-(\d{8})\.csv", filename)
        if match:
            date_key = match.group(1)
            date_fmt = f"{date_key[:4]}-{date_key[4:6]}-{date_key[6:]}"
            
            df = spark.read.option("header", "true").csv(file_path)
            df = df.withColumn("date", lit(date_fmt)).withColumn("date_key", lit(date_key))

            spark.sql("TRUNCATE TABLE master_child")
            df.write.jdbc(jdbc_url, "master_child", mode="append", properties=jdbc_props)

            print(f"✅ Loaded: {filename} into master_child")




### ✅ Step 5: Handle `H_ECOM_ORDER.csv`

This file:

- 📛 Has a fixed name: `H_ECOM_ORDER.csv`  
- 🚫 Requires no transformations  
- ⬇️ Is loaded directly into the `H_ECOM_Orders` table  

```python
    elif filename == "H_ECOM_ORDER.csv":
        df = spark.read.option("header", "true").csv(file_path)

        spark.sql("TRUNCATE TABLE H_ECOM_Orders")
        df.write.jdbc(jdbc_url, "H_ECOM_Orders", mode="append", properties=jdbc_props)

        print(f"✅ Loaded: {filename} into H_ECOM_Orders")


### 📌 Daily ETL Summary

| File Name Example                   | Action Performed                                 | Target Table     |
|------------------------------------|--------------------------------------------------|------------------|
| `CUST_MSTR_20191112.csv`           | ➕ Add `date`, 🧹 truncate, ⬇️ load                | `CUST_MSTR`      |
| `master_child_export-20191112.csv` | ➕ Add `date`, `date_key`, 🧹 truncate, ⬇️ load    | `master_child`   |
| `H_ECOM_ORDER.csv`                 | ✅ Load as-is, 🧹 truncate, ⬇️ load                | `H_ECOM_Orders`  |


- ✅ File detection via filename pattern
- 📅 Date extraction from filenames
- 🧹 Truncate and reload daily
- 💾 Stored in SQL DB via JDBC
