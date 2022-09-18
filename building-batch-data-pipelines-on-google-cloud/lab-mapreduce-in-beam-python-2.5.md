# MapReduce in Beam (Python) 2.5

## Setup
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

- Clone the repository.
```
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
```

## Inspect source
- Open and examine the source file in nano.
- Source file is also available here: https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/courses/data_analysis/lab2/python/is_popular.py
```
cd ~/training-data-analyst/courses/data_analysis/lab2/python
nano is_popular.py
```

- Contemplate the following questions:
    - What custom arguments are defined? --output_prefix and --input.
    - What is the default output prefix? /tmp/output.
    - How is the variable output_prefix in main() set? Through a command-line argument.
    - How are the pipeline arguments such as --runner set? Through command-line arguments.
    - What are the key steps in the pipeline?
        - Read source files.
        - Filter for lines that start with 'import'.
        - Transform to create a list of packages used.
        - Count the number of times each package is used.
        - Find the top 5 most-used packages.
        - Write to output.
    - Which of these steps happen in parallel? GetImports, PackageUse, TotalUse, and Top_5 can be parallelized by Beam.
    - Which of these steps are aggregations? TotalUse and Top_5 are aggregations.

## Execute the pipeline
- Execute the pipeline.
```
python3 ./is_popular.py
```

- Locate and print out contents of the output file.
```
ls -al /tmp
cat /tmp/output-*
```

## Change name of output file
- Use command-line parameters to change the name of the output file.
```
python3 ./is_popular.py --output_prefix=/tmp/myoutput
```

- Locate the new output file and print its contents.
```
ls -lrt /tmp/myoutput*
cat /tmp/myoutput*
```