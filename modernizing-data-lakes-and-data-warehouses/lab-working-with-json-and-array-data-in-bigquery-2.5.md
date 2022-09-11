# Working with JSON and Array data in BigQuery 2.5

- Create a dataset fruit_store
```
bq mk fruit_store
```

## Arrays
- A simple array
```
SELECT ['raspberry', 'blackberry', 'strawberry', 'cherry'] AS fruit_array
```

- Types must be the same in a single array. The next query results in an error.
```
SELECT ['raspberry', 'blackberry', 'strawberry', 'cherry', 1234567] AS fruit_array
```

- Query a table with an array.
- Click JSON to see the results in JSON format.
```
SELECT person, fruit_array, total_cost
FROM `data-to-insights.advanced.fruit_store`;
```

- Load JSON data from GCS to BQ.
```
bq load \
    --source_format=NEWLINE_DELIMITED_JSON \
    --autodetect \
    fruit_store.fruit_details \
    gs://data-insights-course/labs/optimizing-for-performance/shopping_cart.json
```

## Creating arrays
- Explore this table. Note that this one visitId has multiple (111) rows
  associated with it.
```
SELECT
  visitId,
  fullVisitorId,
  date,
  v2ProductName,
  pageTitle
FROM `data-to-insights.ecommerce.all_sessions`
WHERE visitId = 1501570398
ORDER BY date
```

- Use arrays to store v2ProductName and pageTitle. Note that 2 rows are
  returned, one for each day.
```
SELECT
  fullVisitorId,
  date,
  ARRAY_AGG(v2ProductName) AS products_viewed,
  ARRAY_AGG(pageTitle) AS pages_viewed
  FROM `data-to-insights.ecommerce.all_sessions`
WHERE visitId = 1501570398
GROUP BY fullVisitorId, date
ORDER BY date
```

- Count the number of pages and products viewed. 109 pages were viewed by this
  visitor on 20170801.
```
SELECT
  fullVisitorId,
  date,
  ARRAY_AGG(v2ProductName) AS products_viewed,
  ARRAY_LENGTH(ARRAY_AGG(v2ProductName)) AS num_products_viewed,
  ARRAY_AGG(pageTitle) AS pages_viewed,
  ARRAY_LENGTH(ARRAY_AGG(pageTitle)) AS num_pages_viewed
  FROM `data-to-insights.ecommerce.all_sessions`
WHERE visitId = 1501570398
GROUP BY fullVisitorId, date
ORDER BY date
```

- Store only distinct v2ProductName and pageTitle in the arrays. 8 distinct
  pages were viewed by the visitor on 20170801.
```
SELECT
  fullVisitorId,
  date,
  ARRAY_AGG(DISTINCT v2ProductName) AS products_viewed,
  ARRAY_LENGTH(ARRAY_AGG(DISTINCT v2ProductName)) AS distinct_products_viewed,
  ARRAY_AGG(DISTINCT pageTitle) AS pages_viewed,
  ARRAY_LENGTH(ARRAY_AGG(DISTINCT pageTitle)) AS distinct_pages_viewed
  FROM `data-to-insights.ecommerce.all_sessions`
WHERE visitId = 1501570398
GROUP BY fullVisitorId, date
ORDER BY date
```

## Querying ARRAYs
- Query a table with an array. Note BQ visually shows array values in separate rows and
  use gray cells to indicate that said rows are still part of one single row.
```
SELECT *
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_20170801`
WHERE visitId = 1501570398
```

- Access the array hits. Note that the field hits is an ARRAY of STRUCT and
  hits.page is also a STRUCT. This query results in an error.
```
SELECT
  visitId,
  hits.page.pageTitle
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_20170801`
WHERE visitId = 1501570398
```

- The array hits needs to be unpacked with the UNNEST function so that each
  element of the array is unpacked into a separate row. Note that UNNEST() is
  conceptually like a joined table of sorts. It appears in the FROM clause.
```
SELECT DISTINCT
  visitId,
  h.page.pageTitle
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_20170801`
   , UNNEST(hits) AS h
WHERE visitId = 1501570398
LIMIT 10
```

## Inspecting tables with STRUCT fields
- Inspect any of the bigquery-public-data.google_analytics_sample.ga_sessions*
  tables. Note that fields of the type RECORD are STRUCTs. Expand all STRUCTs.
  Note that STRUCTs can contain other STRUCTs. There are 32 of these.

- In the same table, inspect and count the number of ARRAY fields. These will
  have the Mode REPEATED under schema. There are 11.

## Query data with STRUCT fields
- Query STRUCT data. field.* expands all the STRUCT fields in field.
```
SELECT
  visitId,
  totals.*,
  device.*
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_20170801`
WHERE visitId = 1501570398
LIMIT 10
```

## Basics of STRUCT
- Basic STRUCT
```
SELECT STRUCT("Rudisha" as name, 23.4 as split) as runner
```

- STRUCT with an ARRAY field
```
SELECT STRUCT("Rudisha" as name, [23.4, 26.3, 26.4, 26.1] as splits) AS runner
```

## Practice
- Create a dataset racing.
```
bq mk racing
```

- Create a file schema.json containing the schema of a file to import later.
```
echo '[
    {
        "name": "race",
        "type": "STRING",
        "mode": "NULLABLE"
    },
    {
        "name": "participants",
        "type": "RECORD",
        "mode": "REPEATED",
        "fields": [
            {
                "name": "name",
                "type": "STRING",
                "mode": "NULLABLE"
            },
            {
                "name": "splits",
                "type": "FLOAT",
                "mode": "REPEATED"
            }
        ]
    }
]' > schema.json
```

- Import a file from GCS and create a BQ table using the schema in schema.json.
```
bq load \
    --source_format=NEWLINE_DELIMITED_JSON \
    --schema=schema.json \
    racing.race_results \
    gs://data-insights-course/labs/optimizing-for-performance/race_results.json
```

- This query returns only 1 row. race = 800M
```
SELECT * FROM racing.race_results
```

- This query returns an error.
```
SELECT race, participants.name
FROM racing.race_results
```

- Note that participants is of type STRUCT. A table parent with a STRUCT field s
  is like cross-joining table parent with another table s whose schema is the
  same as the STRUCT's. Every row in parent will have a column 
```
SELECT race, p.name
FROM racing.race_results rr     -- alias the table name
CROSS JOIN rr.participants p    -- correlated cross join the STRUCT field participants and alias it
```

- Alternatively, UNNEST() can be used.
```
SELECT race, p.name
FROM racing.race_results rr
   , UNNEST(rr.participants) p
```

- Count the number of racers per race.
```
SELECT COUNT(p.name) AS racer_count
FROM racing.race_results AS r, UNNEST(r.participants) AS p
```

- List the total race times of all racers whose names begin with 'R' and order
  by fastest time first.
```
SELECT
  p.name,
  SUM(split_times) as total_race_time
FROM racing.race_results AS r
CROSS JOIN UNNEST(r.participants) AS p
CROSS JOIN UNNEST(p.splits) AS split_times
WHERE p.name LIKE 'R%'
GROUP BY p.name
ORDER BY SUM(split_times)
```

- Find the racer who had the fastest split time of 23.2.
```
SELECT
  p.name,
  split_time
FROM racing.race_results AS r
, UNNEST(r.participants) AS p
, UNNEST(p.splits) AS split_time
WHERE split_time = 23.2;
```



## Notes
- Arrays are called REPEATED fields in BQ.
- Order elements in an array:
```
ARRAY_AGG(<field> ORDER BY <field>)
```

- Limit number of elements in an array:
```
Limiting ARRAY_AGG(<field> LIMIT 5)
```

- CROSS JOIN a STRUCT
- UNNEST an ARRAY
- CROSS JOIN UNNEST() an ARRAY of STRUCT