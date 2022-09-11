# Loading Taxi Data into Google Cloud SQL 2.5

## Preparing the environment
- Open gcloud console.
- List active account name.
```
gcloud auth list
```

- List project id.
```
gcloud config list project
```

- Export project id and bucket id as environment variables:
```
export PROJECT_ID=$(gcloud info --format='value(config.project)')
export BUCKET=${PROJECT_ID}-ml
echo $PROJECT_ID
echo $BUCKET
```

## Create a Cloud SQL instance
- Create a Cloud SQL instance.
```
gcloud sql instances create taxi --tier=db-n1-standard-1 --activation-policy=ALWAYS
gcloud sql instances list
```

- Set a root password.
```
gcloud sql users set-password root --host % --instance taxi --password Passw0rd
```

- Store IP address of cloud shell as environment variable.
```
export ADDRESS=$(wget -qO - http://ipecho.net/plain)/32
```

- Allow Cloud Shell to access the Cloud SQL instance.
```
gcloud sql instances patch taxi --authorized-networks $ADDRESS
```

- Store IP address of Cloud SQL instance.
```
MYSQLIP=$(gcloud sql instances describe taxi --format="value(ipAddresses.ipAddress)")
echo $MYSQLIP
```

## Login to the Cloud SQL instance and setup
- Login.
```
mysql --host=$MYSQLIP --user=root --password --verbose
```

- Create database.
```
create database if not exists bts;
```

- Select database for use.
```
use bts;
```

- Create tables.
```
drop table if exists trips;
create table trips (
  vendor_id VARCHAR(16),		
  pickup_datetime DATETIME,
  dropoff_datetime DATETIME,
  passenger_count INT,
  trip_distance FLOAT,
  rate_code VARCHAR(16),
  store_and_fwd_flag VARCHAR(16),
  payment_type VARCHAR(16),
  fare_amount FLOAT,
  extra FLOAT,
  mta_tax FLOAT,
  tip_amount FLOAT,
  tolls_amount FLOAT,
  imp_surcharge FLOAT,
  total_amount FLOAT,
  pickup_location_id VARCHAR(16),
  dropoff_location_id VARCHAR(16)
);
```

- Display schema of table.
```
describe trips;
```

- Query the table. Returns no data.
```
select * from trips;
```

- Exit the mysql prompt.
```
exit
```

## Loading data
- Copy CSV files from Google Cloud Storage into local Cloud Shell storage.
```
gsutil cp gs://cloud-training/OCBL013/nyc_tlc_yellow_trips_2018_subset_1.csv trips.csv-1
gsutil cp gs://cloud-training/OCBL013/nyc_tlc_yellow_trips_2018_subset_2.csv trips.csv-2
ls -lt *.csv*
```

- Login to mysql prompt. Note the --local-infile flag. This is to prepare for
  loading of the CSV files downloaded from GCS.
```
mysql --host=$MYSQLIP --user=root  --password --local-infile
```

- Select database for use.
```
use bts;
```

- Load CSV files.
```
LOAD DATA LOCAL INFILE 'trips.csv-1' INTO TABLE trips
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(vendor_id,pickup_datetime,dropoff_datetime,passenger_count,trip_distance,rate_code,store_and_fwd_flag,payment_type,fare_amount,extra,mta_tax,tip_amount,tolls_amount,imp_surcharge,total_amount,pickup_location_id,dropoff_location_id);



LOAD DATA LOCAL INFILE 'trips.csv-2' INTO TABLE trips
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(vendor_id,pickup_datetime,dropoff_datetime,passenger_count,trip_distance,rate_code,store_and_fwd_flag,payment_type,fare_amount,extra,mta_tax,tip_amount,tolls_amount,imp_surcharge,total_amount,pickup_location_id,dropoff_location_id);
```

## Check data loaded
- While still logged in to the Cloud SQL instance, check the number of
  pickup_location_id's. This should return 159.
```
SELECT COUNT(0)
FROM
(
    SELECT DISTINCT(pickup_location_id) FROM trips
) tbl; -- note that this alias is needed
```

- Check minimum and maximum trip distance. Note that minimum trip distanceo of 0
  seems non-sensical.
```
SELECT MAX(trip_distance)
     , MIN(trip_distance)
FROM trips;
```

- How many trips have 0 trip_distance? 155.
```
SELECT COUNT(0)
FROM trips
WHERE trip_distance = 0;
```

- What are fares for these trips with 0 distance?
```
SELECT vendor_id
     , fare_amount
FROM trips
WHERE trip_distance = 0;
```

- Are all fares > 0? No, 14 have negative fares.
```
SELECT COUNT(0)
FROM trips
WHERE fare_amount < 0;
```

- Get count of rows by payment_type.
- Data dictionary: https://www1.nyc.gov/assets/tlc/downloads/pdf/data_dictionary_trip_records_yellow.pdf
```
SELECT payment_type
     , COUNT(0)
FROM trips
GROUP BY payment_type;
```

- Exit.
```
exit
```