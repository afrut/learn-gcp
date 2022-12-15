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
- Dataprep: Web-based interactive tool for transforming data. Good for exploratory data analysis.
- Dataflow: Service for building batch and streaming pipelines using Apache Beam.
- Dataproc: Service for transforming data using Hadoop technologies like Spark.
- Cloud Data Fusion: A low-code/no-code ervice to build ETL data pipelines on top of Dataproc. Based on the open-source project CDAP.

## Compute
- Cloud Functions: Run a function in a serverless and scalable way.
- App Engine: Run an application that can support multiple services in a serverless and scalable way.
- Cloud Run: Run a container in a serverless and scalable way.
- Compute Engine: Managed service for creating virtual machines.
- Google Kubernetes Engine: A managed service for orchestrating containers.

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
- AutoML Tables: A machine learning service for structured data.
- BigQuery ML: Train and request predictions fro models all within BigQuery.
- Dialogflow: Create conversational user interfaces like chat bots.
- Kubeflow: An open source tool for running ML pipelines in Kubernetes.
- Speech-to-Text API converts spoken words to written words.
- Text-to-Speech API converts text to spoken words.
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
- Resource Manager: Manage resources hierarchically; by project, folder, or organization.

# Privacy
- The **Payment Card Industry Data Security Standard (PCI DSS)** is a set of security standards designed to ensure that ALL companies that accept, process, store or transmit credit card information maintain a secure environment
- The **General Data Protection Regulation** is a regulation in EU law on data protection and privacy in the European Union and the European Economic Area.
- The **Health Insurance Portability and Accountability Act of 1996 (HIPAA)** is a federal law that requires the creation of national standards to protect sensitive patient health information from being disclosed.
- The **Children's Online Privacy Protection Act of 1998 (COPPA)** is a US federal law enforced by the Federal Trade Commission that regulates the online collection and use of personal information from children under the age of 13.
- The **Sarbanes-Oxley Act of 2002 (SOX)** is a US regulation on public companies designed to prevent fraudulent accounting practices.

<!-- ------------------------------------------------------------------------------------------ -->
# Notes

<!-- ------------------------------------------------------------ -->
## Storage
- Denormalized relational data models: star and snowflake schemas
- Network models are used to model graph-like structures in graph databases.

### BigQuery
- **BI Engine** caches SQL queries from any source in-memory to improve query performance.
- BigQuery allows querying external data sources. These queries are used to create **external tables**.
    - Allowable formats for creating external tables:
        - Avro
        - CSV
        - newline-delimited JSON
        - Datastore export files
        - Firestore export files
        - ORC
        - Parquet 
- Pre-defined roles:
    - bigquery.dataOwner
    - bigquery.admin
    - bigquery.dataEditor
- Amount of data stored and frequency of refresh can increase costs of materialized views.

### BigTable
- Supports lookup only through a single row key.
- Put incrementing values towards the end of the row key to avoid hot spotting.
- Put columns frequently used together in a **column family**.
- Keep storage utilization per node below 60% for low latency applications.
- Distributes operations based on row keys.
- **HBase** is a column-oriented non-relational database management system that runs on top of Hadoop Distributed File System (HDFS).
- Provides an API compatible with HBase.
- **Hot spotting** occurs when workload is skewed toward a small number of nodes instead of being evenly distributed.
    - Hot spotting is usually caused by keys that are lexicographically similar, like an incrementing key.
    - **Key Visualizer** is a tool for analyzing usage patterns. It is good for troubleshooting hot spotting.
- An **instance** can contain multiple **clusters**. A cluster can contain multiple **nodes**.
- An **app profile** specifies the routing policy that BigTable should use for each request.
    - **Single-cluster routing** routes all requests to 1 cluster in your instance.
    - **Multi-cluster routing** automatically routes requests to the nearest cluster in an instance.
    - **Cluster group routing** sends requests to the nearest available cluster within a cluster group that you specify in the app profile settings.
- **Strong consistency** means that all applications accessing the database will see data in the same state.
- To optimize the performance of BigTable, **replication** read and write operations can be separated.
    - Create one cluster for solely for write and another solely for read operations.
    - Create two app profiles to route read and write traffic to the two separate cluster appropriately.
    - Replication is **eventually consistent**, meaning that for some time, applications may see data in different states.
    - **Read-your-writes consistency** can be achieved for a certain group of applications by routing requests to the same cluster. All other app profiles must route requests to the same cluster. Other clusters can be used for other purposes.
    - Strong consistency can be achieved by configuring for read-your-writes consistency, BUT other clusters cannot be used except for fail over.
    - There is a tradeoff between consistency and latency. Strong consistency incurs higher latency.
- BigTable store up to **10 MB per row**.
- Recommendations are that a table contain **no more than 100 column families**.

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
- **Hot spotting** occurs when operations are skewed towards a few nodes instead of being evenly distributed.
    - Much like BigTable, Cloud Spanner distributes operations based on primary keys.
    - To avoid hot spotting, the following can be employed:
        - Use the hash values of existing primary keys.
        - Bit-reverse sequential values.
    - The following are likely to cause hot spotting:
        - Auto-incrementing/sequential values.
        - Timestamps.
        - Low cardinality attributes.

### Cloud SQL
- Use **Cloud SQL Auth proxy** to connect to Cloud SQL.
- **Read replicas** can be used to improve the performance of the database by off-loading read operations from the primary instance.

### Cloud Storage
- **Transfer Service** is designed to load terabytes of data using scheduled jobs and is well suited for transferring data from other public clouds.
- **Transfer Appliance** is designed for large data loads but requires attaching a storage device to the source system's network and so suitable only when you have physical access to the source system network.
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
    - Nearline for data accessed at most once every 30 days
    - Coldline for data accessed at most once every 90 days
    - Archive for data accessed at most once every 365 days
- `gsutil` is the command-line program to load data into Cloud Storage
- `gsutil rsync` is a command used to sync two buckets/directories.

<!-- ------------------------------------------------------------ -->
## Transformation

### Dataproc
- `FetchFailedException` is an error that occurs when shuffle data is lost, likely due to a node being decommissioned.
- When creating a cluster, `dataproc:dataproc.scheduler.max-concurrent-jobs` can be set to limit the number of concurrent jobs.
- **HDFS** is the Hadoop Distributed File System. It's a file system whose data is stored in a cluster of machines.
- Instead of using HDFS, use Cloud Storage for storing files, which allows data to be accessed even when the cluster is shut down.
- Use no more than 30% of preemptible VMs for secondary workers.
- `roles/dataproc.editor` will provide permissions to stop clusters, initiate workflow templates, and other common user tasks.

### Dataflow
- Because streaming data can be unbounded, a **window** is specified to bound data to be analyzed.
    - sliding (hopping): Every 5 minutes, compute the average in the last hour. Can overlap.
    - fixed (tumbling): Every hour, compute the average. Cannot overlap.
    - session: For the time that a user was active, compute the average. Start and end intervals depend on external events.
- **Watermark** is a time after the end time of a window after which no more late-arriving data is accepted into the window's computation.
- A **PCollection** represents key-value pair data in a distributed fashion.
- A **side input** is an additional input that your `DoFn` can access each time it processes an element in the input PCollection.
- A **custom window** is created using WindowFn functions to implement windows based on data-driven gaps.
- **Flexors** or flexible resource scheduling reduces batch processing costs using scheduling techniques and pre-emptible VMs.
- Metrics:
    - **job/system_lag** is the maximum duration that an item has been waiting in the pipeline.
    - **job/data_watermark_age** is the age of the most recent item that's been fully processed by the pipeline.
    - **job/elapsed_time** is the elapsed time of the pipeline run time.
    - **job/element_count** is the number of items processed in a PCollection for the Read_input and Process_element transforms.

<!-- ------------------------------------------------------------ -->
## Compute

### Compute Engine (CE)
- A **Managed Instance Groups** (MIG) is a group of virtual machines that were created based on a template. Machines are all of the same type. Supports autoscaling.
- An **Unmanaged instance group** is a group of virtual machines that were not created based on a template. Machines can be of different types. Does NOT support autoscaling.

### Google Kubernetes Engine (GKE)
- **deployment.yaml** files are used to configure deployments.

<!-- ------------------------------------------------------------ -->
## Networking
- **DNS A records** associate an IP address with a domain name.

<!-- ------------------------------------------------------------ -->
## Logging

### Cloud Logging
- Only keeps logs for 30 days.
- Logs can be routed to other sinks like Cloud Storage, Pub/Sub, and BigQuery.

<!-- ------------------------------------------------------------ -->
## Access Management

### IAM
- **Service accounts** are identities that represent user-built programs and other cloud services.
- **User accounts** are identities that represent human users.
- A **principal** is an identity or groups of identities that will access and use resources.
- **Permissions** authorize principals to perform operations on resources.
- **Roles** are collections of permissions.
- **Role bindings** grant roles to principals, allowing principals to execute operations authorized by the role's permissions.
- **Policies** are collections of role bindings.
- According to the **principle of least privilege**, grant only the necessary permissions to users and service accounts.
- To authorize a principal to do (x, y, z), a role is created that contains permissions for (x, y, z). This role is then granted to the principal through a policy binding.
- A GCP service Service1 that is not authorized to use another GCP service Service2 indicates that Service1 must be granted the Service Account User role.

### Cloud Key Management Service (KMS)
- Cloud External Key Manager (EKM)

<!-- ------------------------------------------------------------ -->
## AI and Machine Learning
- In **supervised** learning, models use data that has been labeled to predict something.
- In **unsupervised** learning, models find patterns in data without labels.
- **Reinforcement** learning is a technique that trains **agents** to take actions that maximize a reward or minimize consequences.
- **Regression** is a supervised learning problem type that maps data to continuous values.
- **Classification** is a supervised learning problem type that maps data to discrete values.
- **Clustering** is an unsupervised learning problem type that divides the data into groups called clusters. New data is then assigned to these clusters.
- An **instance** is a single row of data with multiple columns.
- In supervised learning, a **target** is a column of data that the model tries to predict.
- A **feature** is a column of data that is used by the model to find patterns/predict the target.
- **Overfitting** is a condition where a model performs well on training data but poorly on test data.
- **Underfitting** is a condition wehere a model performs poorly on training data.
- **Regularization** is a collection of techniques that prevent overfitting.
    - **L2 or Ridge regularization** prevents large coefficients.
    - **L1 or Lasso regularization** can drive the coefficients of features that have no predictive value to 0.
    - **Dropout** is disabling certain neurons in deep learning during training to prevent overfitting.
- **Accuracy** is a classification metric. Of all the predictions, how many are actually true?
- **Precision** is a classification metric. Of all the predictions that are true, how many are actually true?
- **Recall** is a classification metric. Of all the actual instances that are true, how many did the model predict as true?
- **Cloud TPUs** are hardware accelerators used in training deep learning models.
- **Feature crosses** is a way to create synthetic features from existing features (kind of like multiplying two features in linear regression). It is useful when the data set has few features but many instances.
- **Gradient descent** is a way to find the minimum of a function by moving in the direction where the derivative is most negative (steep).
- **Backpropagation** is a widely used algorithm for training neural networks that uses the error and rate of change of the error to calculate weight adjustments. Gradient descent is often used to calculate these weight adjustments.
- Training a Tensorflow model using Compute Engine with a GPU requires the GPU's drivers to be installed. Deep Learning VM images are pre-configured to install GPU drivers.

<!-- ------------------------------------------------------------ -->
## Misc

## Cloud Composer
- **Airflow** is a framework for orchestrating workloads.
- A **task** is an atomic unit of work represented by operators.
- The order in which tasks are performed are specifed in a **Directed Acyclic Graph (DAG)**.
- Python is used to build these DAGs.

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
- A **topic** is an append-only log that producers can publish to and consumers can subscribe to.

### Resource Manager
- Resources in an organization can be restricted to certain locations by setting a Resource Location Restriction policy.
- **constraints/iam.disableServiceAccountKeyCreation** is enabled and that prevents principals from creating user-managed service account keys.
- Enabling **constraints/iam.disableServiceAccountKeyCreation** prevents principals from creating user-managed service account keys.

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
- Compute Engine
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
    - secondary workers
    - gcloud dataproc clusters update --num-secondary-workers
    - gcloud clusters dataproc create --gce-pd-kms-key
- Firestore
    - Indexes
    - Entities
- IAM
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