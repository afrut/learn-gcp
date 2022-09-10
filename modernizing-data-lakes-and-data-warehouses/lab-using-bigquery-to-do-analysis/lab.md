# Using BigQuery to do Analysis
Lab Link:
https://partner.cloudskillsboost.google/course_sessions/1555839/labs/329887

Pin this project to access public data sets: `bigquery-public-data`

## Analyze bike trips
On the search bar, search for `citibike_trips` table and preview data
Find the 50th percentile duration for bike trips between two stations.
Find the number of trips between two stations.
```
SELECT
  MIN(start_station_name) AS start_station_name,
  MIN(end_station_name) AS end_station_name,
  APPROX_QUANTILES(tripduration, 10)[OFFSET (5)] AS typical_duration,
  COUNT(tripduration) AS num_trips
FROM
  `bigquery-public-data.new_york_citibike.citibike_trips`
WHERE
  start_station_id != end_station_id
GROUP BY
  start_station_id,
  end_station_id
ORDER BY
  num_trips DESC
LIMIT
  10
```

Find the total distance traveled by a bike.
```
WITH
  trip_distance AS (
  SELECT
    bikeid,
    ST_DISTANCE(ST_GEOGPOINT(s.longitude, s.latitude), ST_GEOGPOINT(e.longitude, e.latitude)) AS distance
  FROM
    `bigquery-public-data.new_york_citibike.citibike_trips`,
    `bigquery-public-data.new_york_citibike.citibike_stations` AS s,
    `bigquery-public-data.new_york_citibike.citibike_stations` AS e
  WHERE
    start_station_id = s.station_id
    AND end_station_id = e.station_id )
SELECT
  bikeid,
  SUM(distance)/1000 AS total_distance
FROM
  trip_distance
GROUP BY
  bikeid
ORDER BY
  total_distance DESC
LIMIT
  5
```

## Explore weather data set
Open the table ghcn_d.ghcnd_2015 and preview its data.
Return rainfall in mm for a specific weather station for all days in 2015.
```
SELECT
  wx.date,
  wx.value/10.0 AS prcp
FROM
  `bigquery-public-data.ghcn_d.ghcnd_2015` AS wx
WHERE
  id = 'USW00094728'
  AND qflag IS NULL
  AND element = 'PRCP'
ORDER BY
  wx.date
```

## Find correlation between bike trips and rainy days
Tag each day as rainy or not depending on precipitation. Rainy days have
precipitation > 5 mm.

Compute the average number of bike trips on rainy days and non-rainy days.
```
WITH bicycle_rentals AS (
  SELECT
    COUNT(starttime) as num_trips,
    EXTRACT(DATE from starttime) as trip_date
  FROM `bigquery-public-data.new_york_citibike.citibike_trips`
  GROUP BY trip_date
),

rainy_days AS
(
  SELECT
    date,
    (MAX(prcp) > 5) AS rainy
  FROM (
    SELECT
      wx.date AS date,
      IF (wx.element = 'PRCP', wx.value/10, NULL) AS prcp
    FROM
      `bigquery-public-data.ghcn_d.ghcnd_2015` AS wx
    WHERE
      wx.id = 'USW00094728'
  )
  GROUP BY date
)

SELECT
  ROUND(AVG(bk.num_trips)) AS num_trips,
  wx.rainy
FROM bicycle_rentals AS bk
JOIN rainy_days AS wx
ON wx.date = bk.trip_date
GROUP BY wx.rainy
```