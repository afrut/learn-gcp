# Serverless Data Analysis with Dataflow: Side Inputs (Python)

## Setup
- Check if the service account has the Dataflow Developer role.
    - Google Cloud Console > IAM & Admin > IAM > Permissions
    - Look for the principal *-compute@developer.gserviceaccount.com
    - Add the Dataflow Developer role.

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

- Ensure that the Dataflow API is successfully enabled.
```
gcloud services disable dataflow.googleapis.com --force
gcloud services enable dataflow.googleapis.com
```

- Connect to the training VM.
```
gcloud compute ssh $VM --zone=$ZONE
```

- Clone the repository.
```
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
```

- Create some variables.
```
export PROJECT=$(gcloud info --format="value(config.project)")
export BUCKET=$PROJECT
echo "PROJECT=$PROJECT"
echo "BUCKET=$BUCKET"
```

- Create a GCS bucket.
```
gsutil mb -l US gs://$PROJECT
```

## Use BigQuery
- Run the following in BigQuery.
    - Google Cloud Console > BigQuery
    - This query returns the contents of 10 Java files in GitHub in 2016.
```
SELECT
  content
FROM
  `fh-bigquery.github_extracts.contents_java_2016`
LIMIT
  10
```

- Count the number of Java files in the table.
    - Run this analysis in the cloud to take advantage of its distributed/parallel nature.
```
SELECT
  COUNT(*)
FROM
  `fh-bigquery.github_extracts.contents_java_2016`
```

## Explore source code
- Examine source code in nano.
    - Also available here: https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/courses/data_analysis/lab2/python/JavaProjectsThatNeedHelp.py
```
cd ~/training-data-analyst/courses/data_analysis/lab2/python
nano JavaProjectsThatNeedHelp.py
```

- Contemplate the following questions. Refer to dataflow-pipeline.png.
    - Looking at the class documentation at the very top, what is the purpose of this pipeline? It finds Java packages that are popular and need help.
    - Where does the content come from? BigQuery
    - What does the left side of the pipeline do? Find packages that need help.
    - What does the right side of the pipeline do? Find packages that are popular.
    - What does ToLines do? (Hint: look at the content field of the BigQuery result) Can't find ToLines.
    - Why is the result of ReadFromBQ stored in a named PCollection instead of being directly passed to another step? So that it can be split between workers (parallelized)
    - What are the two actions carried out on the PCollection generated from ReadFromBQ? is_popular and needs_help
    - If a file has 3 FIXMEs and 2 TODOs in its content (on different lines), how many calls for help are associated with it? 5. The program counts the number of lines that contains 'FIXME' or 'TODO'.
    - If a file is in the package com.google.devtools.build, what are the packages that it is associated with?
        - com, coim.google, com.google.devtools, com.google.devtools.build
    - popular_packages and help_packages are both named PCollections and both used in the Scores (side inputs) step of the pipeline. Which one is the main - input and which is the side input? help_packages
    - What is the method used in the Scores step? compositeScore
    - What Python data type is the side input converted into in the Scores step? dict

## Execute the pipeline
- Execute the pipeline locally.
    - Note that the program requires either --DirectRunner or --DataFlowRunner
      as an argument depending on whether the program is to be run locally or in
      DataFlow.
```
python3 JavaProjectsThatNeedHelp.py --bucket $BUCKET --project $PROJECT --DirectRunner
```

- Locate the output, download it, and print its contents. Note the timestamp.
```
export OUTPUT=$(gsutil ls gs://$BUCKET/javahelp/Results.csv)
gsutil ls -al $OUTPUT
gsutil cp $OUTPUT Results.csv
cat Results.csv
```

- Execute the pipeline on the cloud. Monitor in DataFlow.
```
python3 JavaProjectsThatNeedHelp.py --bucket $BUCKET --project $PROJECT --DataFlowRunner
```

- Locate the output. Note that the timestamp has changed.
```
export OUTPUT=$(gsutil ls gs://$BUCKET/javahelp/Results.csv)
gsutil ls -al $OUTPUT
gsutil cp $OUTPUT Results.csv
cat Results.csv
```