# Streaming-Data-Processing:-Streaming-Data-Pipelines

## Setup
- Get environment variables.
```
export ACCOUNT=$(gcloud info --format="value(config.account)")
export PROJECT=$(gcloud info --format="value(config.project)")
export VM="training-vm"
export ZONE=$(gcloud compute instances list --filter=$VM --format=json |
    jq -r '.[0].zone' |
    sed "s/^.\+\/\([a-z0-9-]*\)$/\1/")
export REGION=$(echo $ZONE | sed "s/^\([a-z0-9-]\+\)-[a-z]$/\1/")
echo $ACCOUNT
echo $PROJECT
echo $VM
echo $ZONE
echo $REGION
```

- ssh into the virtual machine instance.
```
gcloud compute ssh $VM --zone=$ZONE
```

- Check to see if the virtual machine has done setting up.
    - Look for the following files:
        - bq_magic.sh
        - project_env.sh
        - sensor_magic.sh
```
ls /training
```

- Clone the repository.
```
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
```

- Set environment variables by running this script.
```
source /training/project_env.sh
```

- Create a BigQuery dataset.
```
bq mk demos
```

- Create a GCS bucket.
```
gsutil mb -c standard -l us-central1 $PROJECT
```

## Simulate traffic sensor data into Pub/Sub
- In the terminal with the ssh session to the virtual machine, run:
```
/training/sensor_magic.sh
```

- Open a second terminal and ssh to VM.
```
gcloud compute ssh $VM --zone=$ZONE
```

- In the second terminal, run the following to set environment variables.
```
source /training/project_env.sh
```

## Launch Dataflow pipeline
- Ensure that the Dataflow API is enabled.
```
gcloud services disable dataflow.googleapis.com --force
gcloud services enable dataflow.googleapis.com
```

- Navigate to the script that launches the pipeline.
```
cd ~/training-data-analyst/courses/streaming/process/sandiego
```

- View the script.
    - Also available here on Github: https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/courses/streaming/process/sandiego/run_oncloud.sh
    - Note that this script requires 3 arguments and 1 optional argument:
        - project id
        - bucket name
        - classname (this is a Java file that runs aggregations)
        - options (this is optional)
```
cat run_oncloud.sh
```

- Navigate to the directory containing Java files and examine one.
    - Also available here: https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/courses/streaming/process/sandiego/src/main/java/com/google/cloud/training/dataanalyst/sandiego/AverageSpeeds.java
```
cd ~/training-data-analyst/courses/streaming/process/sandiego/src/main/java/com/google/cloud/training/dataanalyst/sandiego
cat AverageSpeeds.java
```

- Run the Dataflow pipeline.
```
cd ~/training-data-analyst/courses/streaming/process/sandiego
./run_oncloud.sh $DEVSHELL_PROJECT_ID $BUCKET AverageSpeeds
```

## Explore the pipeline
- The Dataflow pipeline pulls messages from the Pub/Sub topic, parses the JSON
  message, produces one main output and writes to BigQuery.
- Examine and monitor the Dataflow pipeline.
    - Google Cloud Console > Dataflow > Jobs > Job Graph
- Examine and monitor the Pub/Sub topic.
    - Google Cloud Console > Pub/Sub > Topics > sandiego > Metrics
- Compare the Dataflow pipeline and the source file AverageSpeeds.Java.
    - Note the PTransform names and the graph.
        - See the PTransform GetMessages. Note that there is no subscription
          created and pulling messages is handled by PubSubIO.readStrings().fromTopic(topic).
    - Note the PTransform TimeWindow.
        - It defines a sliding window of 60 minutes every 30 minutes. Recall
          that the stream is sped up.
    - Note the PTransform BySensor. It groups the data by sensor id.
    - Note the PTransform AvgBySensor. It computes the average by sensor.
    - Note the ToBQRow PTransform. It creates a row to write to BigQuery.
    - Note the BigQueryIO.Write PTransform. It writes (append because of
      WRITE_APPEND) the row to BigQuery.
    - Note that the name of the destination table in BigQuery is demos.average_speeds.

## Examine Dataflow pipeline metrics
- In Google Cloud Console > Dataflow > Jobs > Click on the Job > Job Graph > GetMessages
    - Note System lag. It is the maximum time a message has been waiting for
      processing since it has arrived at this step.
    - Note Elements added. It is the number of elements that have exited this
      step. In this example, it is the number of messages read from Pub/Sub.
- Under Job Metrics, under Autoscaling, note:
    - The number of workers over the course of the job.
    - Click More history. This shows why Dataflow increased/decreased the job.
    
## Check outputs in BigQuery
- Get the most recent 100 records.
```
SELECT *
FROM `demos.average_speeds`
ORDER BY timestamp DESC
LIMIT 100
```

- Get the timestamp of the most recent update.
```
SELECT
MAX(timestamp)
FROM
`demos.average_speeds`
```

- Time-travel to 10 minutes ago. Get the 100 most recent records as of then.
```
SELECT *
FROM `demos.average_speeds`
FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP, INTERVAL 10 MINUTE)
ORDER BY timestamp DESC
LIMIT 100
```

## Restart simulation of traffic sensor data
- Qwiklabs has quotas on message production to Pub/Sub.
- Return to the terminal that simulates producing traffic sensor data to Pub/Sub.
- Press Ctrl-C to interrupt.
- Restart the process. Refer to **Simulate traffic sensor data into Pub/Sub**.

## Use Cloud Monitoring to monitor pipeline metrics
- Cloud Monitoring is a separate service that integrates with Dataflow. It can
  be used to create charts of Dataflow pipeline metrics and create alerts.
- Access Cloud Monitoring.
    - Google Cloud Console > Navigation Menu > Monitoring
    - Wait for the workspace to be provisioned.
- On the left-hand side, click Metrics explorer > Resource & Metric > Select a
  metric > Dataflow Job > Job.
    - This shows available metrics of Dataflow pipelines.
    - Select Data watermark lag > Apply.
    - Note the graph that is creatd on the right side.
- Plot another metric.
    - Under Resource & Metric > Click Dataflow Job - Data Watermark Lag > Reset.
    - Select System lag.
    - see documentation here for Google Cloud metrics: https://cloud.google.com/monitoring/api/metrics_gcp

## Creating alerts
- In Cloud Monitoring, on the left-hand side, select Alerting.
- Create policy > Select a metric > Disable Show only active resources & metrics
- Filter by Dataflow Job > Click Dataflow Job > Job > System Lag > Apply > Next
- Under Configure alert trigger, set
    - Threshold position: Above threshold
    - Threshold value: 5
- Under Advance Options, set Retest window to 1 min
- Click Next
- Click Notification Channels > Manage Notification Channels
- In the new tab, click Add New beside Email and input your email address.
- In the previous tab for creating Alerting policies, click Notification
    Channels and refresh. Select the newly created Email.
- At the bottom of the page, set the policy name to MyAlertPolicy.
- Click Next. Review the policy. Then click Create policy.

## Creating dashboards
- In Cloud Monitoring on the left-hand side,
    - Dashboards > Create dashboard
    - Input a name for New dashboard name
    - Click Line for a line chart
    - Click on the dropdown box under Resource & Metric > Dataflow > Job >
      System Lag > Apply
    - Under Filters > Add Filter and set:
        - Label: project_id
        - Value: your project id
    - Click Done.