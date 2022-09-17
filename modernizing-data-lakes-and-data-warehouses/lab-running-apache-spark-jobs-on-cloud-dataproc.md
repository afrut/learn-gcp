# Running Apache Spark jobs on Cloud Dataproc

## Configure and start a Cloud Dataproc cluster
- Create a cluster.
    - Reference documentation:
        - Versions: https://cloud.google.com/dataproc/docs/concepts/versioning/dataproc-version-clusters
        - Regions and zones: https://cloud.google.com/compute/docs/regions-zones#available
        - Optional components: https://cloud.google.com/dataproc/docs/concepts/components/overview#available_optional_components
```
gcloud dataproc clusters create sparktodp \
    --region=us-west1 --zone=us-west1-a \
    --image-version=2.0-debian10 \
    --enable-component-gateway \
    --optional-components=JUPYTER
```

- Clone a repository and put in home directory ~.
    - `git -C path` runs git as if git were started in path.
```
git -C ~ clone https://github.com/GoogleCloudPlatform/training-data-analyst
```

- Find the Google Cloud Storage bucket used by Cloud Dataproc.
```
export DP_STORAGE="gs://$(gcloud dataproc clusters describe sparktodp --region=us-west1 --format=json | jq -r '.config.configBucket')"
```

- Copy Jupyter notebooks from cloned repository to GCS bucket used by Dataproc.
```
gsutil -m cp ~/training-data-analyst/quests/sparktobq/*.ipynb $DP_STORAGE/notebooks/jupyter
```

- View the notebooks in the Dataproc Cluster.
    - Google Cloud Console > Dataproc > click on cluster > Web Interfaces > Jupyter > GCS > 1_spark.ipynb
    - Note that this notebook uses HDFS as its filesystem. Storage and compute are coupled.

## Decouple storage and compute by using GCS as storage and Dataproc clusters as compute
- Create a GCS bucket to store data files that Dataproc can access.
```
export PROJECT_ID=$(gcloud info --format='value(config.project)')
gsutil mb gs://$PROJECT_ID
```

- Copy files to the newly created bucket.
```
wget https://archive.ics.uci.edu/ml/machine-learning-databases/kddcup99-mld/kddcup.data_10_percent.gz
gsutil cp kddcup.data_10_percent.gz gs://$PROJECT_ID/
```
- Create a copy of Jupyter notebook 01_spark and name it De-couple-storage.

- In spark code, reference the data files in GCS buckets by changing `hdfs://file` to `gs://bucket_name//file`.


## Deploy Spark jobs to Dataproc from Jupyter notebooks
The following steps will create and run a Python script `spark_analysis.py`.

- Create a copy of notebook De-couple-storage.
- Insert a cell above the first cell and paste the following.
    - %%writefile is a Jupyter notebook magic command that writes the contents the cell to a file.
```
%%writefile spark_analysis.py
import matplotlib
matplotlib.use('agg')
import argparse
parser = argparse.ArgumentParser()
parser.add_argument("--bucket", help="bucket for input and output")
args = parser.parse_args()
BUCKET = args.bucket
```

- Add the following to the beginning of the remaining code cells.
    - -a tells Jupyter to append to the file instead of overwriting.
```
%%writefile -a spark_analysis.py
```

- Remove the magic command `%matplotlib inline`.
    - This command instructs the Jupyter notebook to create a matplotlib plot in the cell.
    - we don't want this in our Python script spark_analysis.py.

- Create a new cell at the bottom of the notebook and paste the following:
    - This tells matplotlib to save the plot as a png file.
```
%%writefile -a spark_analysis.py
ax[0].get_figure().savefig('report.png');
```

- Create a new cell at the bottom of the notebook and paste the following:
    - This code deletes all other files in the bucket and uploads the plot created by matplotlib.
```
%%writefile -a spark_analysis.py
import google.cloud.storage as gcs
bucket = gcs.Client().get_bucket(BUCKET)
for blob in bucket.list_blobs(prefix='sparktodp/'):
    blob.delete()
bucket.blob('sparktodp/report.png').upload_from_filename('report.png')
```

- Create a new cell at the bottom of the notebook and paste the following:
    - This code saves the results of the analysis to the GCS bucket.
```
%%writefile -a spark_analysis.py
connections_by_protocol.write.format("csv").mode("overwrite").save(
    "gs://{}/sparktodp/connections_by_protocol".format(BUCKET))
```

- Create a new cell at the bottom of the notebook and paste the following:
    - This code retrieves the bucket name and runs the created Python script spark_analysis.py
    - ! tells Jupyter to run an external CLI command.
```
BUCKET_list = !gcloud info --format='value(config.project)'
BUCKET=BUCKET_list[0]
print('Writing to {}'.format(BUCKET))
!/opt/conda/miniconda3/bin/python spark_analysis.py --bucket=$BUCKET
```

- Create a new cell at the bottom of the notebook and paste the following:
    - List files in the bucket.
```
!gsutil ls gs://$BUCKET/sparktodp/**
```

- Create a new cell at the bottom of the notebook and paste the following:
    - Copy the newly created Python script to GCS storage.
```
!gsutil cp spark_analysis.py gs://$BUCKET/sparktodp/spark_analysis.py
```

## Deploy Spark jobs to Dataproc from Cloud Shell
- Copy the created Python script file.
```
gsutil cp gs://$PROJECT_ID/sparktodp/spark_analysis.py spark_analysis.py
```

- Create a script to submit a PySpark job using the nano text editor.
```
nano submit_onejob.sh
```

- Paste the following into the script. Press CTRL+X then Y and enter key to save and exit.
```
#!/bin/bash
gcloud dataproc jobs submit pyspark \
       --cluster sparktodp \
       --region us-west1 \
       spark_analysis.py \
       -- --bucket=$1
```

- Make the newly created script executable.
```
chmod +x submit_onejob.sh
```

- Execute the script to submit the PySpark job.
```
./submit_onejob.sh $PROJECT_ID
```

## Cleanup
- Delete the cluster.
```
gcloud dataproc clusters delete sparktodp --region=us-west1
```