# Streaming Data Processing: Publish Streaming Data into PubSub

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

- Monitor setup of the vm. Wait for the following files to appear:
    - bq-magic.sh
    - project_env.sh
    - sensor_magic.sh
```
ls -l /training
```

- Clone the repository.
```
git clone https://github.com/GoogleCloudPlatform/training-data-analyst
```

- Store the project id for use later on.
```
export DEVSHELL_PROJECT_ID=$(gcloud config get-value project)
echo $DEVSHELL_PROJECT_ID
```

## Create a Pub/Sub topic and subscription
- Navigate to the following directory.
```
cd ~/training-data-analyst/courses/streaming/publish
```

- Create a topic sandiego.
```
gcloud pubsub topics create sandiego
```

- Publish a simple message.
```
gcloud pubsub topics publish sandiego --message "hello"
```

- Create a subscription mySub1 to topic sandiego.
```
gcloud pubsub subscriptions create --topic sandiego mySub1
```

- Pull the first message that was published to topic sandiego.
    - This should not produce any output because only messages published after a
      subscription's creation are available to that subscription.
```
gcloud pubsub subscriptions pull --auto-ack mySub1
```

- Publish another message and pull a message from the subscription.
```
gcloud pubsub topics publish sandiego --message "hello again"
gcloud pubsub subscriptions pull --auto-ack mySub1
```

- Delete the subscription mySub1.
```
gcloud pubsub subscriptions delete mySub1
```

## Simulate traffic sensor data into Pub/Sub.
- Inspect source code. Also available here: https://github.com/GoogleCloudPlatform/training-data-analyst/blob/master/courses/streaming/publish/send_sensor_data.py
    - Note the simulate function. It accumulates messages in a list, publishes
      them to pub/sub and sleeps for a time.
```
cd ~/training-data-analyst/courses/streaming/publish
nano send_sensor_data.py
```

- Download the traffic simulation data.
```
./download_data.sh
```

- Run send_sensor_data.py
```
./send_sensor_data.py --speedFactor=60 --project $DEVSHELL_PROJECT_ID
```

- Leave this terminal open and running.

## Verify receipt of messages.
- Open another terminal and get the following variables.
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

- ssh into the vm VM.
```
gcloud compute ssh $VM --zone=$ZONE
```

- Navigate to the directory.
```
cd ~/training-data-analyst/courses/streaming/publish
```

- Create a subscription and pull messages to confirm receipt of messages.
```
gcloud pubsub subscriptions create --topic sandiego mySub2
gcloud pubsub subscriptions pull --auto-ack mySub2
```

## Cleanup
- Cancel the subscription mySub2.
```
gcloud pubsub subscriptions delete mySub2
```

- Close the second terminal.
```
exit
```

- Return to the first terminal and hit Ctrl+C and exit it.
```
exit
```