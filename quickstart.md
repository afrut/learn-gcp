# Quickstart for gcloud
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