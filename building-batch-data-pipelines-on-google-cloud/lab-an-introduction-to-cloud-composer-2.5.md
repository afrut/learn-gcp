# An Introduction to Cloud Composer 2.5

## Setup
- Store account and project in environment variables.
```
export ACCOUNT=$(gcloud info --format="value(config.account)")
export PROJECT=$(gcloud info --format="value(config.project)")
export REGION=us-central1
export ZONE=us-central1-a
echo $ACCOUNT
echo $PROJECT
echo $REGION
echo $ZONE
```

- Ensure access to the Kubernetes Engine API.
```
gcloud services disable container.googleapis.com --force
gcloud services enable container.googleapis.com
```

- Ensure access to the Cloud Composer API.
```
gcloud services disable composer.googleapis.com --force
gcloud services enable composer.googleapis.com
```

- Create a Cloud Composer environment.
```
gcloud composer environments create \
    highcpu \
    --location=$REGION \
    --zone=$ZONE \
    --machine-type=n1-highcpu-4
```

- Create a GCS bucket.
```
export BUCKET=$PROJECT
gsutil mb gs://$PROJECT
```

## Learn about Airflow
- Read about:
    - DAG: https://airflow.apache.org/docs/apache-airflow/stable/concepts/dags.html
    - Operator: https://airflow.apache.org/docs/apache-airflow/stable/concepts/operators.html
    - Task: https://airflow.apache.org/docs/apache-airflow/stable/concepts/tasks.html
    - Task Instance: https://airflow.apache.org/docs/apache-airflow/stable/concepts/tasks.html#task-instances

## Defining the workflow
- DAGs are defined in Python files. Airflow will execute these files to
  dynamically build DAGs.
- The DAG to use: https://github.com/GoogleCloudPlatform/python-docs-samples/blob/main/composer/workflows/hadoop_tutorial.py
- Note:
    - The 3 operators used:
        - DataprocClusterCreateOperator
        - DataProcHadoopOperator
        - DataprocClusterDeleteOperator
        - These tasks run sequentially.
    - Name of the DAG: composer_hadoop_tutorial
    - DAG runs once a day
    - The default start date is set to yesterday. Since the start date is
      yesterday, Airflow will start the DAG once it is uploaded.

## Cloud Composer
- In Google Cloud Console, on the right-hand side, select Composer.
- Note:
    - the Airflow web interface URL
    - Kubernetes Engine cluster ID
    - link to the DAGs folder
        - Airflow only runs DAGs in this folder.
- Select the Airflow web interface URL to access Airflow's web UI.
- Select Admin > Variables and create the following variables:
    - gcp_project: value of $PROJECT
    - gcs_bucket: gs://$BUCKET
    - gce_zone: value of $ZONE
- Store some variables:
```
export CENV=$(gcloud composer environments list --locations=$REGION --format=json | jq -r '.[0].name' | sed 's/^.*\/\(.\+\)$/\1/')
export DAGDIR=$(gcloud composer environments describe $CENV --location=$REGION --format="value(config.dagGcsPrefix)")
```

## Upload DAG to GCS
- Copy the DAG file hadoop_tutorial.py from the automatically-created bucket to
  the DAGs folder.
    - In Google Cloud Console > Composer > DAGs folder, check that the file is present
    - In the Airflow UI, note that the DAG composer_hadoop_tutorial is now present.
```
gsutil cp gs://cloud-training/datawarehousing/lab_assets/hadoop_tutorial.py $DAGDIR
```

## Monitor progress in Airflow
- In the airflow UI, go to DAGs > composer_hadoop_tutorial.
    - Tree View shows the status of each task instance.
    - Graph view shows a graph of the DAG as well as task instance status.
- Click Refresh to get the most recent information.
- When the task create_dataproc_cluster is running, go to Google Cloud Console >
  Dataproc to monitor cluster creation and job progress.
- Check the output of the DAG in GCS.
```
gsutil ls gs://$BUCKET/wordcount/20220918-232006/
```