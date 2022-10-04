# Partitioned Tables in Google BigQuery
- Partitioning and clustering are ways of optimizing query performance by
  changing how data is stored.
- Partitioning is an optimization that allows BigQuery to skip scans of data.
  When a table is partitioned on a certain column c, all rows that have the same
  value in column c are stored in a physical block. When a query applies a
  filter based on column c, BigQuery now does not have to scan each and every
  row in that block.
- Clustering is yet another optimization that helps BigQuery optimize filters
  and group bys. When a table is clustered by a set of columns (a, b, c),
  BigQuery sorts the data based on these columns and stores them as blocks. When
  a query filters based on these columns, BigQuery can eliminate scans. When a
  group by on these columns is executed, BigQuery is able to process data faster
  because data is now co-located.
- Try to keep partitions and clusters at around 1GB in size.

## Explore public data
- In Google Cloud Console > BigQuery > Add Data > Pin a project by name
- Input bigquery-public-data.
- Compose a new query. Under More > Query settings > Cache preference > untick
  Use cached results > Save
- Run this query.
    - Bytes processed: 2.69GB
    - Elapsed time: 11sec
    - Slot time consumed: 20sec
    - Bytes shuffled: 482.08MB
```
SELECT id
     , title
     , accepted_answer_id
     , creation_date
     , answer_count 
     , comment_count 
     , favorite_count
     , view_count
     , tags
FROM `bigquery-public-data.stackoverflow.posts_questions`
WHERE creation_date BETWEEN '2018-01-01' AND '2019-01-01';
```

## Create a partitioned and clustered table
- Create a dataset.
```
bq mk mydataset
```

- Create a partitioned table.
```
CREATE OR REPLACE TABLE mydataset.partitioned
PARTITION BY DATE(creation_date) AS
SELECT id
     , title
     , accepted_answer_id
     , creation_date
     , answer_count 
     , comment_count 
     , favorite_count
     , view_count
     , tags
FROM `bigquery-public-data.stackoverflow.posts_questions`
WHERE creation_date BETWEEN '2018-01-01' AND '2019-01-01';
```

- Run the same query on the partitioned table. Note the performance differences.
```
SELECT id
     , title
     , accepted_answer_id
     , creation_date
     , answer_count 
     , comment_count 
     , favorite_count
     , view_count
     , tags
FROM mydataset.partitioned
WHERE creation_date BETWEEN '2018-01-01' AND '2019-01-01';
```

- Create a clustered table.
```
CREATE OR REPLACE TABLE mydataset.clustered
PARTITION BY DATE(creation_date)
CLUSTER BY tabs AS
SELECT id
     , title
     , accepted_answer_id
     , creation_date
     , answer_count 
     , comment_count 
     , favorite_count
     , view_count
     , tags
FROM `bigquery-public-data.stackoverflow.posts_questions`
WHERE creation_date BETWEEN '2018-01-01' AND '2019-01-01';
```

- Run this query on the clustered table. Note the performance differences.
```
SELECT id
     , title
     , accepted_answer_id
     , creation_date
     , answer_count 
     , comment_count 
     , favorite_count
     , view_count
     , tags
FROM mydataset.partitioned
WHERE creation_date BETWEEN '2018-01-01' AND '2019-01-01'
  AND tags = 'android';
```

## More resources
- https://codelabs.developers.google.com/codelabs/gcp-bq-partitioning-and-clustering#0
- https://cloud.google.com/bigquery/docs/partitioned-tables
- https://cloud.google.com/bigquery/docs/clustered-tables