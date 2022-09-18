# A Simple Dataflow Pipeline (Python) 2.5


## Setup
- Ensure that the Dataflow API is successfully enabled.
```
gcloud services disable dataflow.googleapis.com --force
gcloud services enable dataflow.googleapis.com
```

- Get some variables.
```
export VM=training-vm
export PROJECT_ID=$(gcloud info --format="value(config.project)")
export ZONE=$(gcloud compute instances list --filter=$VM --format=json |
    jq -r '.[0].zone' |
    grep -Eo "/[a-z0-9-]+$" |
    sed "s/^\/\(.*\)$/\1/")
export REGION=$(echo $ZONE | sed "s/^\([a-z0-9-]\+\)-[a-z]$/\1/")
echo "PROJECT_ID=$PROJECT_ID"
echo "ZONE=$ZONE"
echo "REGION=$REGION"
```

- ssh into the virtual machine instance.
```
gcloud compute ssh $VM --zone=$ZONE
```

- Create some variables.
```
export PROJECT_ID=$(gcloud info --format="value(config.project)")
export BUCKET=$PROJECT_ID
echo "PROJECT_ID=$PROJECT_ID"
echo "BUCKET=$BUCKET"
```

- Clone a git repository.
```
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
```

- Create a GCS bucket.
```
gsutil mb -l US gs://$PROJECT_ID
``` 

## Inspect source
- View the grep.py file.
```
cd ~/training-data-analyst/courses/data_analysis/lab2/python
nano grep.py
```

- Try to answer the following questions:
    - What files are being read? Java source files.
    - What is the search term? import
    - Where does the output go? /tmp/output
    - What does the transform do? Read .java files.
    - What does the second transform do? Returns all lines that contains the term.
    - Where does its input come from? It comes from the first transform.
    - What does it do with this input? Filters it for the search term.
    - What does it write to its output? All lines containing the search term.
    - Where does the output go? To the 3rd transform.
    - What does the third transform do? Write the results to /tmp/output.

## Execute the pipeline locally on VM
- Run the pipeline.
```
python3 grep.py
```

- Locate the output file and print its contents.
```
ls -al /tmp
cat /tmp/output-*
```

## Executing the pipeline on the cloud
- Copy files from VM to the GCS bucket.
```
gsutil cp ../javahelp/src/main/java/com/google/cloud/training/dataanalyst/javahelp/*.java gs://$BUCKET/javahelp
```

- Use nano to edit grepc.py. Replace PROJECT and BUCKET with the values of PROJECT_ID and BUCKET.
```
nano grepc.py
```

- See the source for grepc.py here:
  https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/courses/data_analysis/lab2/python/grepc.py

- Download the output file from GCS and print the output.
```
gsutil cp $(gsutil ls gs://$BUCKET/javahelp/ | grep "output-*") output
cat output
```