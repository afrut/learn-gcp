# Streaming Data Processing: Streaming Analytics and Dashboards

## Creating a data source
- Access Data Studio: https://datastudio.google.com/
- Login with credentials.
- Reports > Start with a Template > Blank Report
    - Select Continue
    - Select No for every option
- Select Blank Report again.
- Add data to report > Google Connectors > BigQuery
    - Authorize
- My Projects > select your project name > demos > current_conditions > Add > Add to Report

## Create a bar chart using a calculated field
- At the top of the page, click Untitle Report to rename the report.
- Delete the pre-populated tabular report.
- Select Add a chart > Bar > Column chart.
- On the right-hand side
    - Under Dimension, select highway
    - Under Metric
        - Select the X next to record Count to remove it
        - Add metric > latitude
        - Add metric > sensorId
- Select CTD beside sensorId and change it to Count.
- Under name, input vehicles.
- Under Metric, remove latitude.
- The chart shows which highway had the most vehicles detected.
- On the right-hand side, select Style > Show data labels

## Create a bar chart using a custom query
- Add a chart > Bar > Column chart
- On the right-had side, data source, > current_conditions > Add data
- Select BigQuery as a connector > Custom Query > select your project name > input
```
SELECT max(speed) as maxspeed, min(speed) as minspeed,
avg(speed) as avgspeed, highway
FROM `yourprojectid.demos.current_conditions`
group by highway
```

- Select Add.
- Configure the following:
    - Dimension: highway
    - Metric: add minspeed
    - Metric: add maxspeed
    - Metric: add avgspeed
- Under Style > Color by > change colors if desired.

## Viewing query history
- Google Cloud Console > BigQuery
- At the bottom of the page, select Personal History
- This shows the queries executed. On the right-hand side in line with each
  query, click the 3 dots > Show job details for more details