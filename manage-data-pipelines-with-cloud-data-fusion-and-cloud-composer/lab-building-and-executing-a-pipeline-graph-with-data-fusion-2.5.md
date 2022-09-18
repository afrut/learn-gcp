# Building and Executing a Pipeline Graph with Data Fusion 2.5

## Setup
- Get some variables.
```
export ACCOUNT=$(gcloud info --format="value(config.account)")
export PROJECT=$(gcloud info --format="value(config.project)")
export BUCKET=$PROJECT
echo $ACCOUNT
echo $PROJECT
echo $BUCKET
```

- Stop and restart the Cloud Data Fusion API.
```
gcloud services disable datafusion.googleapis.com
gcloud services enable datafusion.googleapis.com
```

- Create a Cloud Data Fusion instance.
    - Google Cloud Console > Navigation Menu > Data Fusion > Instances > Create An Instance
    - Instance name: instance1
    - Edition: Basic
    - Authorization: Grant Permission

- Grant permissions to service account associated with Data Fusion instance.
    - Click on the instance and copy the service account
    - Google Cloud Console > IAM & Admin > IAM > Add
    - Paste the service account as principal
    - Add the Cloud Data Fusion API Service Agent role

## Load data
- Create a GCS bucket and copy some data into it.
```
gsutil mb gs://$BUCKET
gsutil cp gs://cloud-training/OCBL017/ny-taxi-2018-sample.csv gs://$BUCKET
```

- Create a temporary bucket that Cloud Data Fusion can use:
```
gsutil mb gs://$BUCKET-temp
```

- Open Cloud Data Fusion UI.
    - Google Cloud Console > Data Fusion > Instances > Click the instance > View Instance

## Preview data with Wrangler
- Click Wrangler on the Cloud Data Fusion UI.
- In the left-hand panel, click GCS > Cloud Storage Default
- Find the bucket BUCKET and select ny-taxi-2018-sample.csv
- Toggle Use First Row as Header
- Confirm

## Cleaning/Transforming data with Wrangler
- Click the down arrow beside each column name to clean the data for each column.
- Change the data type of the columns:
    - `trip_distance` to Float
    - `total_amount` to Float
    - `pickup_location_id` to String
- Filter the data based on the columns:
    - `trip_distance` > Filter > Custom condition > >0.0

## Creating a pipeline
- Click Create a Pipeline on the upper right-hand corner.
- Select Batch Pipeline.
- Select the Wrangler Node and click Properties.
- Delete the column `extra`.
- Click the Wrangle button to apply more transformations.
- Click Validate on the top-right corner to check for any errors.
- Click the x button on the top-right corner to exit.

## Add a data source
- Create a dataset trips in BigQuery.
```
bq mk --location=US trips
export DATASET=trips
```

- Create a table based on a query.
```
query='SELECT
  zone_id,
  zone_name,
  borough
FROM
  `bigquery-public-data.new_york_taxi_trips.taxi_zone_geom`'
bq --location=US query --destination_table $PROJECT:$DATASET.zone_id_mapping --use_legacy_sql=false $query
```

- Add a source that uses BigQuery.
    - In Cloud Data Fusion Studio, on the left-hand side, under Sources, select BigQuery.
    - On the BigQuery node that appears, click properties.
    - Under Basic, input the following:
        - Reference Name: zone_mapping
        - Dataset: trips
        - Table: zone_id_mapping
        - Temporary Bucket Name: $BUCKET-temp
            - The value of variable BUCKET with -temp appended at the end.
    - Click the Get Schema button.
    - On the top right corner, click Validate then x.

## Join two data sources
- In Cloud Data Fusion Studio, on the left-hand side, under Analytics, click Joiner.
- Join the data in the Wrangler and BigQuery nodes.
    - Click-drag the arrow on the right edge of the Wrangler node and drop on
      the Joiner node.
    - Repeat for the BigQuery node.
- On the Joiner node, click Properties. Set:
    - Join Type: Inner
    - Join Condition > Wrangler: pickup_loation_id
    - Join Condition > BigQuery: zone_id
- Click Get Schema to generate the schema of the join.
- On the right-hand side under Output Schema, delete `pickup_location_id` and `zone_id`.
- In the top-right corner, click Validate and X.

## Store output to BigQuery
- In Cloud Data Fusion Studio, on the right-hand side, under Sink, select BigQuery.
- Connect the Joiner node to the BigQuery sink node BigQuery2 that appears.
- Click Properties in BigQuery2 and configure as follows:
    - Reference Name: bq_insert
    - Dataset: trips
    - Table: trips_pickup_name
    - Temporary Bucket Name: $BUCKET-temp
- Click Validate and X.

## Deploy and run the pipeline
- In Cloud Data Fusion Studio, in the upper-left corner, name the pipeline pipeline1.
- In the upper-right corner, select Deploy.
- In the next screen that appears, select Run.
- Monitor the pipeline.

## Query the results in BigQuery
- Execute a query to see results.
```
query="SELECT tbl.*
FROM \`trips.trips_pickup_name\` tbl"
bq --location=US query --use_legacy_sql=false $query
```