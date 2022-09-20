# Quickstart for gcloud
- gcloud installation: https://cloud.google.com/sdk/docs/install

- Initialize after installation.
```
gcloud init
```

- Login. Uses the browser.
```
gcloud auth login
```

- Store account and project in environment variables.
```
export ACCOUNT=$(gcloud info --format="value(config.account)")
export PROJECT=$(gcloud info --format="value(config.project)")
echo $ACCOUNT
echo $PROJECT
```

- Set project id.
```
gcloud config set project $PROJECT
```

- List all services.
```
gcloud services list
```

- Get the zone of a vm VM.
```
gcloud compute instances list --filter=$VM --format=json |
    jq -r '.[0].zone' |
    sed "s/^.\+\/\([a-z0-9-]*\)$/\1/"
```

- ssh into a virtual machine instance.
```
gcloud compute ssh $VM --zone=$ZONE
```