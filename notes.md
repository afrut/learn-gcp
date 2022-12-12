<!-- ------------------------------------------------------------------------------------------ -->
# Overview of Services
## Ingestion
- Pub/Sub: A global event-streaming service for ingesting data.

## Storage
- Cloud Storage: Scalable object-store for unstructured data.
- BigQuery: SQL-compatible, scalable, column-oriented data warehouse for structured data that is optimized for analytics workloads.
- BigTable: Wide-column, scalable, NoSQL database for semi-strucutured data that supports high-volume low-latency writes and lookup by only a single row key.
- Firestore: Scalable NoSQL document database that supports more complex lookup than just a single row key.
- Cloud SQL: SQL-compatible relational database for transactional workloads that operates in a single region.
- Cloud Spanner: SQL-compatible relational database for transactional workloads that can operate in multiple regions.
- Memorystore: NoSQL in-memory key-value store.

## Transformation
- Dataprep: Web-based interactive tool for transforming data.
- Dataflow: Service for building batch and streaming pipelines using Apache Beam.
- Dataproc: Service for transforming data using Hadoop technologies like Spark.
- Cloud Fusion: TODO

## Compute
- Cloud Functions: Run a function in a serverless and scalable way.
- App Engine: Run an application that can support multiple services in a serverless and scalable way.
- Cloud Run: Run a container in a serverless and scalable way.
- Compute Engine: Managed service for creating virtual machines.

## Access Management
- IAM: Service authenticating identities and authorizing access to various resources.
- Cloud Key Management Service: Manage encryption keys.
- Secret Manager: Store API keys, passwords, certificates and other sensitive data.
- Cloud Identity: TODO

## Logging
- Cloud Monitoring: Monitor and display custom log-based metrics or metrics from different services.
- Cloud Logging: Logging service for services and user-built applications.

# AI and Machine Learning
- AutoML: Automatically train models and tune hyperparameters.
- BigQuery ML: Train and request predictions fro models all within BigQuery.
- Vertex AI: Train, deploy, and request predictions from models. Custom and automatic training through AutoML are available.

# Misc
- Data Catalog: Service for metadata and data discovery.
- Data Loss Prevention (DLP): Service to discover and manage sensitive personal data like names, email, addresses, etc.
- Looker Studio/Data Studio: Build customizable reports and dashboards.
- Virtual Private Cloud (VPC): TODO
- Network Connectivity: TODO
- Cloud Scheduler: Run jobs at regular intervals.
- Cloud Composer: Managed service for orchestrating different workflows using Apache Airflow.
- Cloud CDN: A content delivery network service designed to store copies of data close to end users.

<!-- ------------------------------------------------------------------------------------------ -->
# Notes

<!-- ------------------------------------------------------------ -->
## Storage
- Denormalized relational data models: star and snowflake schemas
- Network models are used to model graph-like structures in graph databases.

### BigQuery
- **BI Engine** caches SQL queries from any source in-memory to improve query performance.
- Amount of data stored and frequency of refresh can increase costs of materialized views.

### BigTable
- Supports lookup only through a single row key.
- Put incrementing values towards the end of the row key to avoid hot spotting.
- Put columns frequently used together in a **column family**.
- **HBase** is a column-oriented non-relational database management system that runs on top of Hadoop Distributed File System (HDFS).
- Provides an API compatible with HBase.
- **Hot spotting** occurs when workload is skewed toward a small number of nodes instead of being evenly distributed.
- Hot spotting is usually caused by keys that are lexicographically similar, like an incrementing key.

### Firestore
- Supports frequently-changing schemas and complex query patterns.
- An **entity** is analogous to a row in relational databases.
- **Kinds** are collections of entities analogous to tables in relational databases.
- **Indexes** are created to support different query patterns.
- Automatically creates atomic value ascending and descending indexes.
- Composite indexes are made up of two or more values and are created manually.
- **MongoDB** is another document database.
- `gcloud datastore export gs://some-bucket --async` creates backups and returns immediately while the backup job runs.

### Cloud Spanner
- Specify parent-child relationships between tables using foreign keys and table interleaving.
- **Interleaving** co-locates parent and child rows which improves performance.

### Cloud SQL
- Use **Cloud SQL Auth proxy** to connect to Cloud SQL.

### Cloud Storage
- Retention policies specify how long files should be kept.
- Retention policy locks prevent changes to the retention period.
- Lifecycle policies can be used to delete data. They are triggered when an object meets certain conditions. These conditions are:
    ```
    age
    createdBefore
    customTimeBefore
    daysSinceCustomTime
    daysSinceNoncurrentTime
    isLive
    matchesStorageClass
    matchesPrefix and matchesSuffix
    noncurrentTimeBefore
    numNewerVersions
    ```
- Storage classes:
    - Standard for frequently-accessed data
    - Nearline for data accessed less than once every 30 days
    - Coldline for data accessed less than once every 90 days
    - Archive for data accessed less than once every 365 days
- `gsutil` is the command-line program to load data into Cloud Storage

<!-- ------------------------------------------------------------ -->
## Transformation

### Dataproc
- `FetchFailedException` is an error that occurs when shuffle data is lost, likely due to a node being decommissioned.
- When creating a cluster, `dataproc:dataproc.scheduler.max-concurrent-jobs` can be set to limit the number of concurrent jobs.

### Dataflow
- Because streaming data can be unbounded, a **window** is specified to bound data to be analyzed.
    - sliding (hopping): Every 5 minutes, compute the average in the last hour. Can overlap.
    - fixed (tumbling): Every hour, compute the average. Cannot overlap.
    - session: For the time that a user was active, compute the average. Start and end intervals depend on external events.
- **Watermark** is a time after the end time of a window after which no more late-arriving data is accepted into the window's computation.
- A **PCollection** represents distributed data.
- A **side input** is an additional input that your `DoFn` can access each time it processes an element in the input PCollection.
- A **custom window** is created using WindowFn functions to implement windows based on data-driven gaps.
- **Flexors** or flexible resource scheduling reduces batch processing costs using scheduling techniques and pre-emptible VMs.

<!-- ------------------------------------------------------------ -->
## Logging

### Cloud Logging
- Only keeps logs for 30 days.
- Logs can be routed to other sinks like Cloud Storage, Pub/Sub, and BigQuery.

<!-- ------------------------------------------------------------ -->
## Access Management
- According to the **principle of least privilege**, grant only the necessary permissions to users and service accounts.

### IAM
- Service accounts are identities that represent user-built programs and other cloud services.
- User accounts are identities that represent human users.

### Cloud Key Management Service (KMS)
- Cloud External Key Manager (EKM)

<!-- ------------------------------------------------------------ -->
## AI and Machine Learning
- **Overfitting** is a condition where a model performs well on training data but poorly on test data.
- **Underfitting** is a condition wehere a model performs poorly on training data.
- **Regularization** is a collection of techniques that prevent overfitting.
    - **L2 or Ridge regularization** prevents large coefficients.
    - **L1 or Lasso regularization** can drive the coefficients of features that have no predictive value to 0.
    - **Dropout** is disabling certain neurons in deep learning during training to prevent overfitting.
- **Accuracy** is a classification metric. Of all the predictions, how many are actually true?
- **Precision** is a classification metric. Of all the predictions that are true, how many are actually true?
- **Recall** is a classification metric. Of all the actual instances that are true, how many did the model predict as true?
- **Backpropagation** is a widely used algorithm for training neural networks that uses the error and rate of change of the error to calculate weight adjustments.
- **Cloud TPUs** are hardware accelerators used in training deep learning models.
- Training a Tensorflow model using Compute Engine with a GPU requires the GPU's drivers to be installed. Deep Learning VM images are pre-configured to install GPU drivers.

<!-- ------------------------------------------------------------ -->
## Misc

### Data Catalog
- Used to tag data to support data discovery.
- Used to store analysis and tagging of PII and sensitive data.
- Can automatically extract metadata from sources like: Cloud Storage, BigQuery, BigTable, Pub/Sub, and Google Sheets.

### Data Loss Prevention
- **PII** stands for Personally Identifiable Information.
- Use re-identification risk analysis (or **risk analysis**) to identify properties or **quasi-identifiers**, like age or postal code, that can re-identify a person even in redacted data.

### Data Studio/Looker Studio
- Used to build custom reports and dashboards.
- Live data sources are connections to the source data without importing any data into Looker.
- Extracted data sources are snapshots that are stored in an in-memory cache and can provide better performance.
- File upload data sources import CSV data into Looker.
- Blended data sources combine data from multiple sources in the same visualization.

### Pub/Sub
- At least once delivery guarantee.
- Can deliver data out of order and can deliver duplicates. Use in conjunction with Dataflow if you need data to be in order and to deduplicate.
- subscription/num_undelivered_messages is a metric that indicates how well subscribers are keeping up with data being ingested.
- Define a schema during topic creation to ensure  to ensure messages are written with a standard structure.
- Supports Protocol Buffer (ProtoBuf) and Avro formats for schema definition.
- **Thrift** is an alternative to ProtoBuf.
- **Parquet** is an open-source column-oriented format used in Hadoop systems.

<!-- ------------------------------------------------------------------------------------------ -->
# Useful Links
- Data engineering code demos: https://github.com/GoogleCloudPlatform/training-data-analyst/tree/master/courses/data-engineering/demos

<!-- ------------------------------------------------------------------------------------------ -->
# Learn About
- AI and Machine Learning
    - Difference between TPU and CPU
- BigQuery
    - Nested and repeated fields (pre-joining and co-locating)
- BigTable
    - Hbase
    - Nodes, tablets, hotspotting
- Cloud Functions
    - gcloud functions deploy
- Cloud Logging
    - Cloud Audit Logs
- Cloud SQL
    - strong consistency
    - ACID
    - read replicas
- Cloud Spanner
    - strong consistency
    - ACID
    - secondary indexes
- Cloud Storage
    - Retention policy, retention policy lock, lifecycle policy
    - Storage Transfer Service vs Transfer Appliance
- Compute Engine
    - unmanaged vs managed instance groups (MIGs)
    - shielded VMs
- Cloud Key Management Service (KMS)
    - CMEK vs CSEK
- Cloud CDN
- Data Loss Prevention
    - infotypes
- Data Studio
    - prefetch caching
- Dataflow
    - Streaming Engine
    - Dataflow shuffle
    - gcloud dataflow sql query
- Dataproc
    - pre-emptible VMs
    - node group, sole-tenant node groups
    - autoscaling
    - high-availability SSD
    - gcloud dataproc clusters update --num-secondary-workers
    - gcloud clusters dataproc create --gce-pd-kms-key
- Firestore
    - Indexes
    - Entities
- IAM
    - Authentication: Identities and principals
    - Authorization: Roles and permissions
    - public/private service account keys
- Memorystore
- Network Connectivity
    - Cloud VPN
    - Cloud Interconnect
        - Dedicated Interconnect
        - Partner Interconnect
    - Cloud Router
    - Partner Interconnect
- Pub/Sub
    - pull vs push subscription