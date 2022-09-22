# Streaming Data Processing: Streaming Data Pipelines into Bigtable

## Setup
- Get some variables.
```
export ACCOUNT=$(gcloud info --format="value(config.account)")
export PROJECT=$(gcloud info --format="value(config.project)")
export VM=training-vm
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

- ssh into the vm.
```
gcloud compute ssh $VM --zone=$ZONE
```

- Wait for the vm to finish setup. Run the following command look for these files:
    - bq-magic.sh
    - project_env.sh
    - sensor_magic.sh
```
ls /training
```

- Clone the repository.
```
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
```

- Set environment variables.
```
source /training/project_env.sh
```

- Prepare HBase quickstart files.
```
cd ~/training-data-analyst/courses/streaming/process/sandiego
./install_quickstart.sh
```

## Stream traffic sensor data into Pub/Sub
- Start streaming data to Pub/Sub. This will send 1 hour's worth of data in 1 minute.
```
/training/sensor_magic.sh
```

- In another terminal instance, open a 2nd ssh session to the vm.
```
gcloud compute ssh $VM --zone=$ZONE
source /training/project_env.sh
```

## Launch Dataflow pipeline
- Ensure that the Dataflow API is enabled.
```
gcloud services disable dataflow.googleapis.com --force
gcloud services enable dataflow.googleapis.com
```

- Examine the script. It starts a Dataflow pipeline.
```
cd ~/training-data-analyst/courses/streaming/process/sandiego
nano run_oncloud.sh
```

- Create a Bigtable instance.
```
cd ~/training-data-analyst/courses/streaming/process/sandiego
./create_cbt.sh
```

- Run the Dataflow pipeline to stream data into Bigtable.
```
cd ~/training-data-analyst/courses/streaming/process/sandiego
./run_oncloud.sh $DEVSHELL_PROJECT_ID $BUCKET CurrentConditions --bigtable
```

- Examine the pipeline in Google Cloud Console > Dataflow.

## Query Bigtable
- In the 2nd terminal instance, run
```
cd ~/training-data-analyst/courses/streaming/process/sandiego/quickstart
./quickstart.sh
```

- At the HBase prompt, input:
    - This will retrieve two rows from the Bigtable table. It may take a few minutes.
    - Note the format of (column, timestamp, value).
```
scan 'current_conditions', {'LIMIT' => 2}
```

- Retrieve the lane:speed column for 10 rows from start row to end row.
```
scan 'current_conditions', {'LIMIT' => 10, STARTROW => '15#S#1', ENDROW => '15#S#999', COLUMN => 'lane:speed'}
```

- Exit the Hbase prompt.
```
quit
```

## Cleanup
- Delete the Bigtable instance.
```
cd ~/training-data-analyst/courses/streaming/process/sandiego
./delete_cbt.sh
```

- Replace job_id with the actual job_id.
```
gcloud dataflow jobs list
gcloud dataflow jobs cancel job_id
```

- In the first terminal instance, press Ctrl+C to stop streaming data to Pub/Sub.

- Delete the demos dataset in BigQuery.
```
bq rm demos
```