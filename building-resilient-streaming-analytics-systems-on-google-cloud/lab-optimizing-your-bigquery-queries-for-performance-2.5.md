# Optimizing your BigQuery Queries for Performance 2.5

## Setup
- Store account and project in environment variables.
```
export ACCOUNT=$(gcloud info --format="value(config.account)")
export PROJECT=$(gcloud info --format="value(config.project)")
echo $ACCOUNT
echo $PROJECT
```

## Disabling query caching in Google Cloud Console
BigQuery caches query results. When tuning, turn caching off to properly
experiment with performance characteristics. More > Query settings > Untick Use
cached results.

## Minimizing I/O

### Select only needed columns
- Most overhead for simple queries is due to I/O, not computation. Computing the
  variance of a column will be almost just as fast as computing the mean of a
  column because most of the overhead is due to I/O.
- BigQuery is a columnar store. Select only the columns that you need. Don't
  mindlessly SELECT *.
- Run this query in Google Cloud Console > BigQuery. Note that:
    - Job Information > Bytes processed: 371.84MB
    - Execution Details:
        - Elapsed time: 332 ms
        - Slot time consumed: 2 sec
        - Bytes shuffled: 468 B
```
SELECT
  bike_id,
  duration
FROM `bigquery-public-data`.london_bicycles.cycle_hire
ORDER BY duration DESC
LIMIT 1
```

- Run this query. Note:
    - Job Information > Bytes processed: 2.59 GB
    - Execution Details:
        - Elapsed time: 613 ms
        - Slot time consumed: 12 sec
        - Bytes shuffled: 3.33 KB
```
SELECT *
FROM `bigquery-public-data`.london_bicycles.cycle_hire
ORDER BY duration DESC
LIMIT 1
```

- The first query consumed less resources simply by having less columns.

### Re-use needed columns when grouping and filtering
- Sometimes it is possible to filter and group rows using columns included in
  the SELECT clause. This also reduces the number of columns read.
- This query filters and groups based on `start_station_id` and
  `end_station_id`, both of which are not included in SELECT.
    - Bytes processed: 1.86 GB
    - Elapsed time: 2 sec
    - Slot time consumed: 1 min 12 sec
    - Bytes shuffled: 1.04 GB
```
SELECT
  MIN(start_station_name) AS start_station_name,
  MIN(end_station_name) AS end_station_name,
  APPROX_QUANTILES(duration, 10)[OFFSET (5)] AS typical_duration,
  COUNT(duration) AS num_trips
FROM `bigquery-public-data`.london_bicycles.cycle_hire
WHERE start_station_id != end_station_id
GROUP BY
  start_station_id,
  end_station_id
ORDER BY num_trips DESC
LIMIT 10
```

- This query is identical to the first but groups by `start_station_name` and
  `end_station_name`, both of which are already included in SELECT.
- At the time of writing, this query is not significantly faster but does
  process less data:
    - Bytes processed: 1.5 GB
    - Elapsed time: 2 sec
    - Slot time consumed: 1 min
    - Bytes shuffled: 1.013.82 MB
```
SELECT
  start_station_name,
  end_station_name,
  APPROX_QUANTILES(duration, 10)[OFFSET(5)] AS typical_duration,
  COUNT(duration) AS num_trips
FROM `bigquery-public-data`.london_bicycles.cycle_hire
WHERE start_station_name != end_station_name
GROUP BY
  start_station_name,
  end_station_name
ORDER BY num_trips DESC
LIMIT 10
```

### Reduce the number of expensive computations
- When expensive computations need to be performed, sometimes it is possible to
  execute this computation on a smaller set of data before joining to a larger
  set of data.
- This query joins the more transactional `cycle_hire` table to the more static
  `cycle_stations` table. Then it executes `ST_Distance`, an expensive
  computation, on the already-joined `cycle_stations` data, which is much larger
  after the join.
```
WITH
trip_distance AS (
    SELECT
        bike_id,
        ST_Distance(ST_GeogPoint(s.longitude,s.latitude),
            ST_GeogPoint(e.longitude,e.latitude)
        ) AS distance
    FROM `bigquery-public-data`.london_bicycles.cycle_hire,
        `bigquery-public-data`.london_bicycles.cycle_stations s,
        `bigquery-public-data`.london_bicycles.cycle_stations e
    WHERE start_station_id = s.id AND end_station_id = e.id
)

SELECT
  bike_id,
  SUM(distance)/1000 AS total_distance
FROM trip_distance
GROUP BY bike_id
ORDER BY total_distance DESC
LIMIT 5
```

- This query first executes the expensive `ST_Distance` computation on the
  smaller `cycle_stations` table. Then it joins that result set to the much
  larger `cycle_hire` table. This effectively executes `ST_Distance` on a
  smaller result set before joining to a larger result set.
```
WITH
stations AS (
    SELECT
        s.id AS start_id,
        e.id AS end_id,
    ST_Distance(
        ST_GeogPoint(s.longitude,s.latitude),
        ST_GeogPoint(e.longitude,e.latitude)
    ) AS distance
    FROM `bigquery-public-data`.london_bicycles.cycle_stations s,
        `bigquery-public-data`.london_bicycles.cycle_stations e
)

, trip_distance AS (
    SELECT
        bike_id,
        distance
    FROM `bigquery-public-data`.london_bicycles.cycle_hire,
        stations
    WHERE start_station_id = start_id
      AND end_station_id = end_id
)

SELECT
    bike_id,
    SUM(distance)/1000 AS total_distance
FROM trip_distance
GROUP BY bike_id
ORDER BY total_distance DESC
LIMIT 5
```

## Caching Query Results
BigQuery caches results of queries so that if the same query were submitted
within approximately 24 hours, the results retrieved from the cache instead of
computing the query again. Cached results are much faster and do not incur
charges.

BigQuery caching behaviour:
- BigQuery compares the string submitted as a query to determine if it has
  cached results. So any string differences, like whitespaces, between otherwise
  similar queries will cause BigQuery to recompute the query.
- BigQuery never caches results if:
    - Queries are never cached if they have non-deterministic behaviour such as
    CURRENT_TIMESTAMP and RAND.
    - The underlying view or table has changed, even when the rows/columns of
      interest remain the same.
    - The table has a streaming buffer, even when there is no new data in the
      streaming buffer.
    - The query uses DML statements.
    - The query uses external data sources.

### Materializing frequently-executed queries
- Sometimes, multiple queries execute the same subquery A. To avoid recomputing
  the result set of A, it can be executed once and have its results stored in a
  table or materialized view. Then, other queries can access the table instead
  of recomputing A. This strategy incurs I/O cost to save computing.
- Run the following query. Note that it has a WITH clause.
    - Bytes processed: 1.68 GB
    - Elapsed time: 4 sec
    - Slot time consumed: 2 min 25 sec
    - Bytes shuffled: 1.14 GB
```
WITH
typical_trip AS (
    SELECT
        start_station_name,
        end_station_name,
        APPROX_QUANTILES(duration, 10)[OFFSET (5)] AS typical_duration,
        COUNT(duration) AS num_trips
    FROM `bigquery-public-data`.london_bicycles.cycle_hire
    GROUP BY start_station_name, end_station_name
)

SELECT
  EXTRACT (DATE FROM start_date) AS trip_date,
  APPROX_QUANTILES(duration / typical_duration, 10)[OFFSET (5)] AS ratio,
  COUNT(*) AS num_trips_on_day
FROM `bigquery-public-data`.london_bicycles.cycle_hire AS hire
JOIN typical_trip AS trip
    ON hire.start_station_name = trip.start_station_name
    AND hire.end_station_name = trip.end_station_name
    AND num_trips > 10
GROUP BY trip_date
HAVING num_trips_on_day > 10
ORDER BY ratio DESC
LIMIT 10
```

- Create a table with the results of a frequently-executed query.
    - Get some variables.
    ```
    export ACCOUNT=$(gcloud info --format="value(config.account)")
    export PROJECT=$(gcloud info --format="value(config.project)")
    echo $ACCOUNT
    echo $PROJECT
    ```

    - Get target dataset's location.
    ```
    export DATASET_LOCATION=$(bq show --format=json 'bigquery-public-data:london_bicycles' | jq -r '.location')
    ```

    - Create a dataset. Make it the same location as the original dataset.
    ```
    bq --location=$DATASET_LOCATION mk "$PROJECT:mydataset"
    ```

    - Create a table using the query from the WITH clause. Run this in Google
      Cloud Console.
    ```
    CREATE OR REPLACE TABLE mydataset.typical_trip AS
    SELECT
        start_station_name,
        end_station_name,
        APPROX_QUANTILES(duration, 10)[OFFSET (5)] AS typical_duration,
        COUNT(duration) AS num_trips
    FROM `bigquery-public-data`.london_bicycles.cycle_hire
    GROUP BY start_station_name, end_station_name
    ```

    - Execute this query. Note:
        - Bytes processed: 1.72 GB
        - Elapsed time: 2 sec
        - Slot time consumed: 58 sec
        - Bytes shuffled: 129.12 MB
    ```
    SELECT
        EXTRACT (DATE FROM start_date) AS trip_date,
        APPROX_QUANTILES(duration / typical_duration, 10)[OFFSET(5)] AS ratio,
        COUNT(*) AS num_trips_on_day
    FROM `bigquery-public-data`.london_bicycles.cycle_hire AS hire
    JOIN mydataset.typical_trip AS trip
        ON hire.start_station_name = trip.start_station_name
        AND hire.end_station_name = trip.end_station_name
        AND num_trips > 10
    GROUP BY trip_date
    HAVING num_trips_on_day > 10
    ORDER BY ratio DESC
    LIMIT 10
    ```

    - The second query ran faster than the first and had less bytes shuffled
      although the number of bytes processed is roughly similar. The savings
      increase when multiple queries use the same table that was made from the
      results of a subquery.
    - A down-side to this pattern is that when new data is added to
      `cycle_hire`, the data in `typical_trip` is not refreshed. So,
      `typical_trip` will have to be re-created at some certain frequency.

### BI Engine
BI Engine helps speed up queries in BI settings (eg. Google Data Studio). It
automatically stores relevant pieces of data in memory and uses a query
processor that is optimized for in-memory data to deliver query results to BI
solutions much faster. It functions as a kind of cache with in-memory query
processing.

Memory for BI Engine can be reserved under BigQuery Admin Console. Make sure to
reserve memory in the same region as the dataset being queried.

## Joining efficiently

### Denormalization
- Joining is an expensive computation. If many other queries depend on a common
  result set that is computed with a join, a table can be created with this
  result set so that subsequent queries query this table without having to
  repeat the join. This pre-joining is called denormalization. Like reducing
  expensive computation, this strategy trades increased I/O and storage for
  reduced computation.
```
CREATE OR REPLACE TABLE mydataset.london_bicycles_denorm AS
SELECT
  start_station_id,
  s.latitude AS start_latitude,
  s.longitude AS start_longitude,
  end_station_id,
  e.latitude AS end_latitude,
  e.longitude AS end_longitude
FROM `bigquery-public-data`.london_bicycles.cycle_hire AS h
JOIN `bigquery-public-data`.london_bicycles.cycle_stations AS s
ON h.start_station_id = s.id
JOIN `bigquery-public-data`.london_bicycles.cycle_stations AS e
ON h.end_station_id = e.id
```

### Avoiding unnecessary self-joins
- The following query retrieves the top 5 most common names for male babies in
  2015 in the state of Massachusetts.
    - Make sure query is running in `us (multiple regions in United States)`
      - More > Query settings > Additional settings > Data location
```
SELECT
  name,
  number AS num_babies
FROM `bigquery-public-data`.usa_names.usa_1910_current
WHERE gender = 'M'
  AND year = 2015
  AND state = 'MA'
ORDER BY num_babies DESC
LIMIT 5
```

- The following is the same query for female names.
```
SELECT
  name,
  number AS num_babies
FROM `bigquery-public-data`.usa_names.usa_1910_current
WHERE gender = 'M'
  AND year = 2015
  AND state = 'MA'
ORDER BY num_babies DESC
LIMIT 5
```

- This query naively attempts to take the top 5 most common baby names assigned
  to both genders. It uses two scans of the data and joins the two result sets.
    - Note that this query returns results that are incorrect because it cross
      joins across year and state columns.
    - 
```
WITH
male_babies AS (
    -- One full scan of the data
    SELECT name
         , number AS num_babies
    FROM `bigquery-public-data`.usa_names.usa_1910_current
    WHERE gender = 'M'
)

, female_babies AS (
    -- Another full scan of the data
    SELECT name
         , number AS num_babies
    FROM `bigquery-public-data`.usa_names.usa_1910_current
    WHERE gender = 'F'
)

, both_genders AS (
    -- Join two result set of both scans
    SELECT name
         , SUM(m.num_babies) + SUM(f.num_babies) AS num_babies
         , SUM(m.num_babies) / (SUM(m.num_babies) + SUM(f.num_babies)) AS frac_male
    FROM male_babies AS m
    JOIN female_babies AS f
    USING (name)
    GROUP BY name
)

-- Get top 5 most common names assigned to both male and female babies.
SELECT *
FROM both_genders
WHERE frac_male BETWEEN 0.3 AND 0.7
ORDER BY num_babies DESC
LIMIT 5
```

- The previous query can be improved by reducing the number of full scans.
```
WITH
all_babies AS (
    -- First and only full scan. This subquery removes the year and state
    -- dimensions, thereby reducing the number of rows. This is similar
    -- to an aggregation of sorts without having to use group by.
    SELECT name
         , SUM(IF(gender = 'M', number, 0)) AS male_babies
         , SUM(IF(gender = 'F', number, 0)) AS female_babies
    FROM `bigquery-public-data.usa_names.usa_1910_current`
    GROUP BY name
)

, both_genders AS (
    -- This subquery select from an aggregated result set, which has less rows.
    -- This is not a full scan.
    SELECT name
        , (male_babies + female_babies) AS num_babies
        , SAFE_DIVIDE(male_babies, male_babies + female_babies) AS frac_male
    FROM all_babies
    WHERE male_babies > 0
      AND female_babies > 0
)

SELECT *
FROM both_genders
WHERE frac_male BETWEEN 0.3 AND 0.7
ORDER BY num_babies DESC
LIMIT 5
```

### Reduce the data being joined
- If a join needs to be used, try to reduce the result sets to be joined.
```
WITH
all_names AS (
    -- Full scan with aggregation using group by.
    SELECT name
         , gender
         , SUM(number) AS num_babies
    FROM `bigquery-public-data`.usa_names.usa_1910_current
    GROUP BY name, gender 
)

, male_names AS (
    -- Selecting from reduced result set.
    SELECT name
         , num_babies
    FROM all_names
    WHERE gender = 'M'
)

, female_names AS (
    -- Selecting from reduced result set.
    SELECT name
         , num_babies
    FROM all_names
    WHERE gender = 'F'
)

, ratio AS (
    -- Join two reduced result sets.
    SELECT name
         , (f.num_babies + m.num_babies) AS num_babies
         , m.num_babies / (f.num_babies + m.num_babies) AS frac_male
    FROM male_names AS m
    JOIN female_names AS f
    USING (name)
)

SELECT *
FROM ratio
WHERE frac_male BETWEEN 0.3 AND 0.7
ORDER BY num_babies DESC
LIMIT 5
```

### Use a window functions instead of self-join
- This query returns the start and end times of a bike rental.
```
SELECT bike_id, start_date, end_date
FROM `bigquery-public-data`.london_bicycles.cycle_hire
LIMIT 5
```

- Suppose that the problem is to find the average time that a bike is in the
  station, not out on rental. One way to do this is with a join. This is an
  example where there is a relationship between rows. This query takes a long
  time and is not practical.
```
WITH
start_sub_end AS (
    -- This subquery returns start_date - end_date > 0 for every bike_id
    SELECT t.bike_id
         , t.start_date
         , t.end_date
    FROM `bigquery-public-data`.london_bicycles.cycle_hire t

    -- Self-join, more like a cross self-join
    JOIN `bigquery-public-data`.london_bicycles.cycle_hire t2
      ON t.bike_id = t2.bike_id           -- For rows with matching bike_id
      AND TIMESTAMP_DIFF(t.start_date, t2.end_date, SECOND) > 0  -- Subtract every start_date with every end_date and return only the ones that are positive
),

-- The smallest positive start_date - end_date is the time the bike
-- was at the station between rentals.
time_between_rentals AS (
    SELECT bike_id
        , MIN(start_date - end_date) AS time_at_station
    FROM start_sub_end t
    GROUP BY bike_id
)

SELECT bike_id
     , AVG(time_at_station)
FROM time_between_rentals
GROUP BY bike_id
LIMIT 5
```

- However, the relationship between rows is such that given a bike_id, when
  rows are ordered by start_date ascending, then the time that the bike was at
  the station between rentals is simply the start_date OF THE NEXT ROW minus the
  end_date of the current row. The LAG window function easily accomplishes this
  more easily.
```
WITH
unused AS (
    SELECT bike_id
         , start_station_name
         , start_date
         , end_date
         , TIMESTAMP_DIFF(start_date, LAG(end_date) OVER (PARTITION BY bike_id ORDER BY start_date), SECOND) AS time_at_station
    FROM `bigquery-public-data`.london_bicycles.cycle_hire
)

SELECT start_station_name
     , AVG(time_at_station) AS unused_seconds
FROM unused
GROUP BY start_station_name
ORDER BY unused_seconds ASC
LIMIT 5
```

### Join with a smaller, pre-computed table
- Suppose that the problem is now to find the two stations between which bikers
  travel the fastest. One way to do it is the query below. But note that it is
  joining a transactional type table `cycle_hire`, which is huge, with more
  static table `cycle_stations`, then executing an expensive computation
  `ST_DISTANCE`.
```
WITH
denormalized_table AS (
    SELECT start_station_name
        , end_station_name
        , ST_DISTANCE(ST_GeogPoint(s1.longitude,s1.latitude), ST_GeogPoint(s2.longitude,s2.latitude)) AS distance
        , duration
    FROM `bigquery-public-data`.london_bicycles.cycle_hire AS h
    JOIN `bigquery-public-data`.london_bicycles.cycle_stations AS s1
    ON h.start_station_id = s1.id
    JOIN `bigquery-public-data`.london_bicycles.cycle_stations AS s2
    ON h.end_station_id = s2.id
)

, durations AS (
    SELECT start_station_name
        , end_station_name
        , MIN(distance) AS distance
        , AVG(duration) AS duration
        , COUNT(*) AS num_rides
    FROM denormalized_table
    WHERE duration > 0
      AND distance > 0
    GROUP BY start_station_name, end_station_name
    HAVING num_rides > 100
)

SELECT start_station_name
     , end_station_name
     , distance
     , duration
     , duration/distance AS pace
FROM durations
ORDER BY pace ASC
LIMIT 5
```

- Note that in the previous query, the data for for cycle stations is only used
  with ST_DISTANCE. In fact, it is possible to compute the distance between each
  station with the smaller table `cycle_stations` first, then joining to
  `cycle_hire`. See below.
```
WITH
distances AS (
    -- Execute ST_DISTANCE on cycle_stations before the join.
    SELECT
           a.id AS start_station_id
         , a.name AS start_station_name
         , b.id AS end_station_id
         , b.name AS end_station_name
         , ST_DISTANCE(ST_GeogPoint(a.longitude,a.latitude)
         , ST_GeogPoint(b.longitude,b.latitude)) AS distance
    FROM `bigquery-public-data`.london_bicycles.cycle_stations a
    CROSS JOIN `bigquery-public-data`.london_bicycles.cycle_stations b
    WHERE a.id != b.id
)

, durations AS (
    -- Reduce the size of the result set by aggregation before joining.
    SELECT start_station_id
         , end_station_id
         , AVG(duration) AS duration
         , COUNT(*) AS num_rides
    FROM `bigquery-public-data`.london_bicycles.cycle_hire
    WHERE duration > 0
    GROUP BY start_station_id, end_station_id
    HAVING num_rides > 100
)

-- Join the two tables.
SELECT start_station_name
     , end_station_name
     , distance
     , duration
     , duration/distance AS pace
FROM distances
JOIN durations
USING (start_station_id, end_station_id)
ORDER BY pace ASC
LIMIT 5
```

## Avoiding overloading a worker
- BigQuery is a distributed system. Sometimes, queries can specify jobs that may
  overload a worker. This is can cause the worker to run out of memory and
  result in an unfinished job. Strategies to mitigate this usually involving
  partitioning the data into small enough sizes for multiple workers to process.
  The memory available to each worker is in the order of 1 GB.

### Large sorts
- Suppose that the problem is to create a column that assigns an incremental
  number to rentals ordered by end_date. One way to do this might be:
```
SELECT rental_id
     , ROW_NUMBER() OVER(ORDER BY end_date) AS rental_number
FROM `bigquery-public-data.london_bicycles.cycle_hire`
ORDER BY rental_number ASC
LIMIT 5
```

- But if the data set is large enough, the worker will run out of memory and
  will be unable to complete the sort. To solve this, rows within a given day
  can first be sorted, then the rows can be sorted by date in the final sort.
  Note that the data is being partitioned by day.
```
WITH
rentals_on_day AS (
    SELECT rental_id
         , end_date
         , EXTRACT(DATE FROM end_date) AS rental_date
    FROM `bigquery-public-data.london_bicycles.cycle_hire`
)

SELECT rental_id
     , rental_date
     , ROW_NUMBER() OVER(PARTITION BY rental_date ORDER BY end_date) AS rental_number_on_day
FROM rentals_on_day
ORDER BY rental_date ASC
       , rental_number_on_day ASC
LIMIT 5
```

### Data skew
- Data skew usually occurs during group bys and array_agg(). When executing a
  group by on a column, sometimes, one dimension value may have much much more
  data than other values. These are then assigned to a worker, thus overloading
  the worker. Suppose the problem is to aggregate rows with ARRAY_AGG() grouped
  by time zone. The shape of the data is such that most authors live in only a
  few time zones, thus overloading a worker.
```
SELECT repo_name
     , ARRAY_AGG(STRUCT(author
                       ,committer
                       ,subject
                       ,message
                       ,trailer
                       ,difference
                       ,encoding)
                 ORDER BY author.date.seconds)
FROM `bigquery-public-data.github_repos.commits`,
UNNEST(repo_name) AS repo_name
GROUP BY repo_name
```

- To solve this, data can be aggregated over both time zone and another key
  (repo_name). If further aggregation over only time zone is needed, then
  another query can be written to aggregate over repo_name. In a way, this
  involves partitioning the data into smaller chunks by adding another key to
  the aggregation.
```
SELECT repo_name
     , author.tz_offset
     , ARRAY_AGG(STRUCT(author
                       ,committer
                       ,subject
                       ,message
                       ,trailer
                       ,difference
                       ,encoding)
                 ORDER BY author.date.seconds)
FROM `bigquery-public-data.github_repos.commits`,
UNNEST(repo_name) AS repo_name 
GROUP BY repo_name, author.tz_offset
```

## Approximate aggregate functions
- There are approximate versions of most aggregation functions. These versions
  faster and more efficient if an error of about 1% is tolerable.
    - See https://cloud.google.com/bigquery/docs/reference/standard-sql/approximate_aggregate_functions

## Notes
- Documentation on query performance and optimization: https://cloud.google.com/bigquery/docs/query-plan-explanation?_ga=2.16336950.-1205130924.1664036952 
- Introduction to optimizing query performance: https://cloud.google.com/bigquery/docs/best-practices-performance-overview
- Control costs in BigQuery: https://cloud.google.com/bigquery/docs/best-practices-costs
- In general I/O and storage are cheaper than compute cost. However, it is best
  to run experiments to determine the performance characteristics of the data set.