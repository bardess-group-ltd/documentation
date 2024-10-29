# Databricks Streaming with Kafka and Checkpoints

## Overview
This document describes how to set up a real-time streaming pipeline in Databricks using Spark Structured Streaming. The pipeline reads data from Kafka, decodes Avro messages using a schema registry, and writes the data to a Delta Lake table. The process leverages checkpointing to ensure data consistency, fault tolerance, and exactly-once processing semantics. It is designed to be flexible and reusable for different environments (e.g., Production, UAT).

## Prerequisites
- **Databricks Workspace**: A configured workspace with necessary permissions.
- **Kafka Cluster**: A Kafka cluster accessible from Databricks.
- **Schema Registry**: A Confluent Schema Registry to handle AVRO schema.
- **Databricks Secrets**: Stored secrets (API keys, passwords) in Databricks Secret Scope.
- **Unity Catalog**: Enabled for managing Delta Lake tables.

## Architecture
The streaming pipeline comprises:

1. **Kafka Data Source**: Streams data from a Kafka topic.

2. **Schema Registry**: Decodes AVRO-encoded messages.

3. **Checkpointing**: Uses checkpoints to track streaming progress and recover from failures.

4. **Delta Lake Sink**: Writes the processed data into a Delta Lake table in Unity Catalog.


## Main Components

### 1. **Configuration and Initialization**

- **Environment Handling**:
  
    - The script accepts an environment parameter (`Production` or `UAT`) to load environment-specific settings (Kafka topic, secrets scope).
  
    - Reads a JSON file to fetch environment-specific variables like database schema, topic prefix, Kafka servers, and schema registry address.

- **Secure Secrets Management**:
  
    - Uses `dbutils.secrets.get()` to fetch sensitive information (API keys, secrets) from the configured secret scope, ensuring security.

- **Dynamic Configuration**:
  
    - The script dynamically configures the Kafka topic, table name, and checkpoint path based on the environment and topic provided.

#### Sample Code Snippet:
```python
environment = dbutils.widgets.get("environment")
file_name = "tc_uc_prod_kafka" if environment == "Production" else "tc_uc_uat_kafka"
with open(f"/Workspace/Shared/kafka-load-streaming/environment_variables/{file_name}.json", "r") as file:
    result = json.load(file)
kafka_bootstrap_servers = result.get("kafka_bootstrap_servers")
schema_registry_address = result.get("schema_registry_address")
```

### 2. **Setting Up the Spark Session**
- The Spark session is configured to use Kafka and Delta Lake. Required packages (`kafka-clients`, `delta`) are specified during session creation.

#### Sample Code Snippet:
```python
spark = SparkSession.builder \
    .appName("StreamingApp") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.jars.packages", "org.apache.kafka:kafka-clients:2.6.0") \
    .getOrCreate()
```

### 3. **Kafka Configuration**
- **Security Protocol**: Configures Kafka to use SASL_SSL for secure communication.
- **Offsets**:
    - **Starting Offsets**: Uses `"earliest"` to consume data from the earliest available offset if the consumer group is new.
    - **Checkpoint Directory**: The `checkpointLocation` parameter is used to store the streaming state, ensuring fault tolerance.
- **Consumer Group**: A unique consumer group ID is generated for each streaming job run to isolate processing.

#### Sample Code Snippet:
```python
group_id = result.get("group_id") + '-streaming-' + str(uuid.uuid4())
checkpoint_path = f"/Workspace/Shared/kafka-load-streaming/checkpoint/{environment}_{topic}"
```

### 4. **Streaming Data from Kafka**
- **AVRO Decoding**: Uses the Confluent schema registry to decode AVRO messages into a structured DataFrame.
- **Schema Registry Integration**: Provides the schema registry address and credentials for decoding messages.

#### Sample Code Snippet:
```python
df_new = (spark
    .readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", kafka_bootstrap_servers)
    .option("subscribe", topic)
    .option("kafka.security.protocol", security_protocol)
    .option("kafka.sasl.mechanism", "PLAIN")
    .option("startingOffsets", "earliest")
    .option("checkpointLocation", checkpoint_path)
    .load()
    .select(from_avro(col("value"), schemaRegistryAddress=schema_registry_address).alias("value"))
)
```

### 5. **Using Checkpoints for Fault Tolerance**
- **Checkpointing**: The checkpoint directory is specified using `.option("checkpointLocation", checkpoint_path)`. It stores:
    - Offsets for the Kafka stream.
    - Progress of the stream, including watermark and processing state.
- **Recovery**: In case of job failure or restart, Spark Structured Streaming will use the information in the checkpoint directory to resume processing from the last processed offset, ensuring exactly-once semantics.

### 6. **Writing to Delta Lake**
- **Delta Format**: The streaming data is written to a Delta table using the `.writeStream.format("delta")` API. The use of Delta Lake ensures ACID transactions, data versioning, and efficient query capabilities.
- **Output Mode**: Uses `"append"` mode to continuously append new records to the Delta table.

#### Sample Code Snippet:
```python
df_new.writeStream \
    .format("delta") \
    .outputMode("append") \
    .option("checkpointLocation", checkpoint_path) \
    .table(table_path)
```

## Checkpointing Details
- **Checkpoint Path**: A unique path is assigned to each streaming job to store checkpoint data.
- **Data Resilience**: Checkpointing enables Spark to keep track of the offsets processed and helps resume from the last successful point in case of a failure, avoiding data loss or reprocessing.
- **Storage**: The checkpoint files are stored in the specified DBFS location (e.g., `/Workspace/Shared/kafka-load-streaming/checkpoint/`).

## Security Considerations
- **Secrets Management**: The use of Databricks secrets for sensitive credentials (API keys, secrets) ensures that credentials are not hard-coded in the script, improving security.
- **Kafka SASL_SSL**: Secure communication with Kafka using SASL_SSL and JAAS configuration ensures data in transit is encrypted.

## Error Handling and Recovery
- **Fail on Data Loss**: `.option("failOnDataLoss", "false")` is used to handle cases where the data stream might encounter data loss (e.g., message deletion in Kafka). This setting ensures the stream continues processing despite missing data.
- **Dynamic Consumer Group**: A new, unique consumer group is created for each run to avoid offset conflicts.

## Best Practices
- **Unique Checkpoint Path**: Always use a unique checkpoint path for each streaming job or topic to avoid offset conflicts and enable proper fault tolerance.
- **Use Delta Lake**: Delta Lake provides ACID transactions, data versioning, and improved performance for both streaming and batch processing.
- **Configure Secrets**: Store sensitive information such as API keys and secrets in Databricks Secret Scope and use `dbutils.secrets.get()` to access them securely.

## Example Usage
- **Starting the Stream**: Execute the notebook with the required environment and topic as widgets. The script will dynamically configure the Kafka connection, read the stream, decode messages using Avro, and write the data to Delta Lake.
- **Resuming After Failure**: If the streaming job fails or stops, it will resume from the last processed offset using the data in the checkpoint directory.

## Summary
This generic Databricks streaming pipeline:
- Consumes data from Kafka topics using secure configurations.
- Uses a schema registry to decode Avro-encoded messages.
- Writes the processed data into Delta Lake tables.
- Implements checkpointing for fault tolerance and exactly-once processing.

By configuring dynamic variables, secure secrets, and using Delta Lake, this pipeline is flexible, reusable, and suitable for various streaming scenarios in different environments (Production, UAT).