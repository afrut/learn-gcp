# Loading Taxi Data into Google Cloud SQL 2.5

## Upload data from local machine
- Download this CSV file: https://storage.googleapis.com/cloud-training/OCBL013/nyc_tlc_yellow_trips_2018_subset_1.csv
- Open the BigQuery Console
- Create a new dataset nyctaxi under your project
- Create a new table 2018trips under nyctaxi
    - Select upload file and browse the local machine to select the file
    - Specify CSV as file format
    - Under schema, tick Auto Detect
- Check number of rows in Details tab of nyctaxi.2018trips. It should be 10018.
- Top 5 most expensive trips of the year
```
SELECT *
FROM nyctaxi.2018trips
ORDER BY fare_amount DESC
LIMIT  5
```

## Ingest data in Google Cloud Storage to BigQuery
- Open Cloud Shell
- Load file from GCS to BQ
    - --source_format=CSV specifies the format of the file.
    - --autodetect asks BQ to infer the schema.
    - --noreplace asks BQ to append data to the existing nyctaxi.2018trips table.
```
bq load \
    --source_format=CSV \
    --autodetect \
    --noreplace  \
    nyctaxi.2018trips \
    gs://cloud-training/OCBL013/nyc_tlc_yellow_trips_2018_subset_2.csv
```

- Check number of rows in Details tab. It should be 20024.

## Create new tables from SQL queries and DDL statements
- Create a table for only January trips
```
CREATE TABLE nyctaxi.january_trips AS
SELECT *
FROM nyctaxi.2018trips
WHERE EXTRACT(Month FROM pickup_datetime) = 1
```

- Find the longest distance traveled in the month of January
```
SELECT *
FROM nyctaxi.january_trips
ORDER BY trip_distance DESC
LIMIT 1
```