# 📘 ETL Project: Load Files from Data Lake to SQL

## ✅ Objective
You have 3 types of CSV files stored in a **Data Lake folder**, and your goal is to:

- 🔄 Load them into their respective **database tables**
- 🧹 Perform **truncate-and-load** operation **daily**
- 📅 Extract **date information from the file name** and use it in the data for certain files

---

## 🗂️ File Types and Rules

| File Example                      | Load Target Table     | Transformation Needed                                           |
|----------------------------------|------------------------|------------------------------------------------------------------|
| `CUST_MSTR_20191112.csv`         | `CUST_MSTR`            | ➕ Add `date` column from filename → `2019-11-12`                |
| `master_child_export-20191112.csv` | `master_child`       | ➕ Add `date` → `2019-11-12`<br>➕ Add `date_key` → `20191112`    |
| `H_ECOM_ORDER.csv`               | `H_ECOM_Orders`        | ✅ Load **as-is** (no transformation)                            |

---
# 🧰 ETL Pipeline in 5 Steps

---

## ✅ Step 1: Spark Session & Setup

In this step, I:

* 🚀 Start my Spark session
* 📁 Define the data lake path
* 🔌 Set up JDBC configuration for loading into the SQL database

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import lit
import re

# Start Spark session
spark = SparkSession.builder.appName("Daily_ETL_Pipeline").getOrCreate()

# Define Data Lake base path
data_lake_path = "/mnt/datalake/container/"  # Replace with actual path

# JDBC Configuration for SQL Server
jdbc_url = "jdbc:sqlserver://<server>:<port>;databaseName=<dbname>"
jdbc_props = {
    "user": "<username>",
    "password": "<password>",
    "driver": "com.microsoft.sqlserver.jdbc.SQLServerDriver"
}
```

---

## ✅ Step 2: List All Files and Loop Through Them

Here, I:

* 🔍 List all the CSV files from the container
* 📄 Loop through each file and apply logic depending on its name

```python
files = dbutils.fs.ls(data_lake_path)

for file in files:
    filename = file.name
    file_path = file.path
```

---

## ✅ Step 3: Handle `CUST_MSTR_YYYYMMDD.csv` Files

In this step:

* I identify files that start with `CUST_MSTR_`
* I extract the date from the file name and convert it to `YYYY-MM-DD`
* I add this as a new `date` column
* Then I truncate the `CUST_MSTR` table and load the data

```python
    if filename.startswith("CUST_MSTR_") and filename.endswith(".csv"):
        match = re.search(r"CUST_MSTR_(\d{8})\.csv", filename)
        if match:
            date_raw = match.group(1)
            date_fmt = f"{date_raw[:4]}-{date_raw[4:6]}-{date_raw[6:]}"
            
            df = spark.read.option("header", "true").csv(file_path)
            df = df.withColumn("date", lit(date_fmt))

            spark.sql("TRUNCATE TABLE CUST_MSTR")
            df.write.jdbc(url=jdbc_url, table="CUST_MSTR", mode="append", properties=jdbc_props)

            print(f"✅ Loaded: {filename} into CUST_MSTR")
```

---

## ✅ Step 4: Handle `master_child_export-YYYYMMDD.csv` Files

Here:

* I detect files that start with `master_child_export-`
* I extract both `date` (`YYYY-MM-DD`) and `date_key` (`YYYYMMDD`) from the filename
* I add both columns to the DataFrame
* I truncate the `master_child` table and load the data

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
```

---

## ✅ Step 5: Handle `H_ECOM_ORDER.csv` File

In this final step:

* I load the `H_ECOM_ORDER.csv` file
* I don't apply any transformation
* I just truncate and load it into the `H_ECOM_Orders` table

```python
    elif filename == "H_ECOM_ORDER.csv":
        df = spark.read.option("header", "true").csv(file_path)

        spark.sql("TRUNCATE TABLE H_ECOM_Orders")
        df.write.jdbc(jdbc_url, "H_ECOM_Orders", mode="append", properties=jdbc_props)

        print(f"✅ Loaded: {filename} into H_ECOM_Orders")
```

---

## 📌 Daily ETL Summary

| File Name Example                  | What I Did                                            | Target Table    |
| ---------------------------------- | ----------------------------------------------------- | --------------- |
| `CUST_MSTR_20191112.csv`           | ➕ Added `date`, 🧹 truncated old data, ⬇️ loaded file | `CUST_MSTR`     |
| `master_child_export-20191112.csv` | ➕ Added `date` & `date_key`, 🧹 truncated, ⬇️ loaded  | `master_child`  |
| `H_ECOM_ORDER.csv`                 | ✅ Loaded as-is, 🧹 truncated old data                 | `H_ECOM_Orders` |

---

## 🧠 What I Did in Simple Words

* 📁 I checked all files inside my Data Lake folder
* 🧠 I used filename patterns to detect file types
* 📅 Extracted dates from filenames wherever needed
* ➕ Added extra columns (`date`, `date_key`) where required
* 🧹 Cleared (truncated) old data from tables
* 💾 Loaded fresh data into SQL tables

---

**Author:** Prakash Pandey
**LinkedIn:** [https://www.linkedin.com/in/prakash-pandey-2827522b1/](https://www.linkedin.com/in/prakash-pandey-2827522b1/)

