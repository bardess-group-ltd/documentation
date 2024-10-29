### Technical Documentation

#### Overview

This code consists of two main parts:

1.  **Variable Setup**: Reads and sets up various configurations required for connecting to Kafka, handling security credentials, and reading schema information.
    
2.  **Kafka Streaming and Avro Decoding**: Establishes a Kafka streaming job using Spark Structured Streaming to read, decode Avro messages using a schema registry, and write the data into Delta Lake tables in Databricks.
    

### Part 1: Variable Setup and Configuration

#### Code Breakdown

1.  **Import Required Libraries**:
    
    *   Imports utilities from PySpark, Spark SQL functions, and the DBUtils for secret management.
        
    *   uuid and json are used to handle unique IDs and JSON data respectively.
        
2.  **Retrieve the Environment**:
    
    *   Uses dbutils.widgets.get("environment") to fetch the current environment setting. This could be "Production" or "UAT".
        
    *   Depending on the environment, it sets the appropriate file name and secret scope for accessing sensitive information (e.g., API keys).
        
3.  **Read Environment Variables**:
    
    *   Reads a JSON file (.json) located in /Workspace/Shared/kafka-load-streaming/environment\_variables/.
        
    *   Extracts and stores various configuration details such as db\_audit\_schema, db\_source\_schema, topic\_prefix, schema\_registry\_address, kafka\_bootstrap\_servers, etc., from this JSON file.
        
4.  **Retrieve Secrets**:
    
    *   Uses Databricks' dbutils.secrets.get() to fetch sensitive credentials (api\_key, api\_secret, sra\_api\_key, sra\_api\_secret) from the configured secret scopes.
        
5.  **Determine Kafka Topic and Table Name**:
    
    *   Topic is pushed down from the job and retrieved by using a widget.

    *   Queries a table (data\_dictionary) to get the corresponding table name for the specified topic. If the topic is not found, the topic name itself is used as the table name.
        
    *   Constructs the topic name by combining the topic\_prefix and the topic.
        
6.  **Set Kafka Consumer Group ID**:
    
    *   Generates a unique Kafka consumer group ID using uuid.uuid4() to prevent offset conflicts across different runs of the streaming job.
        
7.  **Set Checkpoint Path**:
    
    *   Defines a unique checkpoint path in the DBFS to store progress information of the streaming query.
        
8.  **Define Kafka Offsets**:
    
    *   The offset will set from the last saved checkpoint if the offset folders exist with data. 

    *   Uses "earliest" as the starting offset if this is the first run ever. 
        

#### Key Configurations

*   **Checkpointing**: Uses checkpointLocation to maintain state information about the Kafka stream, ensuring exactly-once processing.
    
*   **Secrets Management**: Fetches sensitive information from Databricks secret scopes, making it secure.
    
*   **Dynamic Configurations**: Reads environment-specific settings (e.g., production or UAT) from JSON files, allowing the same script to be used in multiple environments.
    

### Part 2: Kafka Streaming and Decoding

#### Code Breakdown

1.  **Create Spark Session**:
    
    *   Sets up a Spark session with required configurations (Java environment, Delta Lake extension, and Kafka package).
        
2.  **Configure Schema Registry**:
    
    *   Defines options for accessing the schema registry using the sra\_api\_key and sra\_api\_secret retrieved from Databricks secrets.
        
3.  **Define Kafka Parameters**:
    
    *   Sets Kafka security configurations (sasl\_mechanism, security\_protocol) for SSL communication with the Kafka brokers.
        
    *   Constructs sasl\_jaas\_config using the Kafka credentials to facilitate authentication.
        
4.  **Set Up Kafka Stream**:
    
    *   Configures the Kafka stream using spark.readStream:
        
        *   **Kafka Servers**: Uses kafka.bootstrap.servers to specify the Kafka broker address.
            
        *   **Topics**: Subscribes to the topic, defined using topic\_prefix and the topic from the widget.
            
        *   **Security**: Configures security options (sasl\_mechanism, sasl\_jaas\_config) for authentication.
             
        *   **Offsets**: Uses "earliest" as the starting offset or the offset from the checkpoint to start at the last known offset.
            
        *   **Error Handling**: Uses .option("failOnDataLoss", "false") to avoid stopping the stream in case of data loss.
            
    *   Uses from\_avro to decode Avro data from Kafka messages using the schema registry.
        
5.  **Write Stream to Delta Table**:
    
    *   Writes the streaming data to a Delta table in Unity Catalog using outputMode("append") and specifying the checkpointLocation to maintain state.
        
6.  **Audit Table Update**:
    
    *   Creates a simple DataFrame containing the topic and group ID information.
        
    *   Appends this information to an audit table to keep track of active streams and their associated group IDs.
        

#### Notes

*   **AVRO Decoding**: Uses from\_avro to decode Avro-encoded messages from Kafka using the schema registry, which is vital for schema evolution and message deserialization.
    
*   **Delta Lake**: Uses Delta Lake's outputMode("append") to write data to a specified table, supporting ACID transactions and efficient data management.
 
*   **Check Point**: Uses checkpoints to manage data ingestion. This helps maintain data accuracy and integrity incase of corruption