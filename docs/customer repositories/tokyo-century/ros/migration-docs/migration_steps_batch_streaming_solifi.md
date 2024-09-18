# Migration from Batch Processing to Streaming in Databricks

## Introduction

This document outlines the steps required to migrate from batch data processing to streaming data processing in Databricks. The current batch processing setup for Solifi’s data from Kafka takes around 3–4 minutes to complete. By transitioning to a streaming data pipeline, we aim to process this data in near real-time. More about the process can be found [here](./docs/customer repositories/tokyo-century/ros/code-workflow-docs/ros_source_streaming.md)


### Pre-migration Step

Before initiating the migration process, ensure the following:
- **Production data is not streaming**, and **no activities** are currently taking place in production.
- Confirm that all ongoing batch processes have been completed or stopped.

---

## Migration Steps

### Step 1: Run Initial Setup Queries

1. Access the following [Databricks SQL Editor Link](https://dbc-cb7ebf60-fb25.cloud.databricks.com/sql/editor/0f7a7e7c-6913-4887-93cf-e01d92a22f1b?o=6184472344017424) and run the setup queries for the migration of history data into source tables.

---

### Step 2: Migrate Source to Bronze (dlt_src_to_brnz)

1. **Update the environment variable** to production.
   - Ensure that the environment configuration reflects production settings.
2. **Run the history load to bronze**:
   - Execute the Delta Live Table (DLT) pipeline `dlt_src_to_brnz` to load historical batch data into the **bronze table**.
   - This step is crucial for initializing the bronze layer with past data.

---

### Step 3: Start the Bronze Pipeline (dlt_bronze_pipeline)

1. **Update the environment variable** to production, similar to Step 2.
2. **Start the bronze pipeline**:
   - Once the tables in `dlt_src_to_brnz` have been created, initiate the `dlt_bronze_pipeline` to process the incoming streaming data into the bronze layer.
   - Monitor the pipeline to ensure the tables are being populated with real-time data.

---

### Step 4: Start ROS Streaming Topics to Source

1. **Update the environment variable** to production.
   - Ensure the environment variables and settings are switched to production.
2. **Start the streaming job for ROS topics**:
   - Start the job responsible for streaming ROS topics from Kafka into the source tables.
   - This step transitions the workflow from batch-based processing to continuous streaming.

---

## Post-migration Validation

Once the migration steps are complete, it is important to validate the streaming setup to ensure everything is functioning as expected:

1. **Monitor the streaming pipelines**:
   - Ensure the streaming pipelines are processing data without significant lag or errors.
   - Check the performance metrics of the pipelines to identify any bottlenecks.
   
2. **Data Quality Check**:
   - Validate the quality of the data being ingested into the bronze layer and ensure that no data loss or corruption occurred during the migration.
   
3. **Update Monitoring Dashboards**:
   - Update the existing monitoring dashboards to reflect the streaming data architecture instead of the batch process.
   
4. **Test Failover and Recovery**:
   - Run tests to ensure that the streaming pipelines can handle unexpected failures and automatically recover without data loss.

---

## Conclusion

The migration from batch processing to streaming in Databricks enables near real-time data processing, significantly reducing the data processing delay from minutes to seconds. Ensure continuous monitoring and validation of the streaming pipelines for optimal performance.

---

## References

- [Databricks Streaming Documentation](https://docs.databricks.com/spark/latest/structured-streaming/index.html)
- [Delta Live Tables Documentation](https://docs.databricks.com/workflows/delta-live-tables/index.html)

---

### Appendix

- **Environment Variables**: Ensure all environment variables are set properly for production use.
- **ROS Topics**: The list of streaming topics configured for the migration.
- **Monitoring Tools**: List the tools used to monitor the streaming pipelines.
