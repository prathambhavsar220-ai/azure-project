Azure Data Factory Incremental Data Ingestion
Overview
This project uses Azure Data Factory to incrementally copy data from Azure SQL Database into Azure Data Lake Storage Gen2.
Instead of copying an entire table during every run, the pipelines use a saved date or timestamp called a watermark to select newer records. The extracted data is stored as Parquet files in the bronze storage layer.
The repository includes a pipeline for processing one table and another for processing multiple tables sequentially.
Technologies Used
- Azure Data Factory
- Azure SQL Database
- Azure Data Lake Storage Gen2
- Apache Parquet
- Azure Logic Apps
- GitHub
How It Works
1. Read the previous watermark from a JSON file in the data lake.
2. Select source records whose tracking column is greater than the watermark.
3. Copy those records into a timestamped Parquet file.
4. If data was read, query the source table’s maximum tracking value and update the watermark.
5. Otherwise, delete the current output file.
For example, a table using updated_at as its tracking column loads records with an updated_at value later than the saved checkpoint.
This is watermark-based ingestion. It does not capture source deletions, and it captures updates only when the tracking column advances.
Pipelines
Incremental_Ingestion
Processes a single SQL table using these parameters:
- schema: SQL schema, such as dbo.
- table: Source table, such as DimUser.
- cdc_clumn: Tracking column, such as updated_at. Use this exact spelling.
- from_date: Optional starting value. An empty string uses the saved watermark.
The pipeline reads the checkpoint, copies qualifying records, and either updates the checkpoint or removes the empty output.
Incremental_Loop
Processes multiple tables through a sequential ForEach loop.
The default configuration includes:
- DimUser, tracked using updated_at.
- DimTrack, tracked using updated_at.
- DimDate, tracked using date.
- DimArtist, tracked using updated_at.
- FactStream, tracked using stream_timestamp.
Its loop_input parameter contains the schema, table, tracking column, and optional starting date for each table. The tracking-column field is named cdc_col in this pipeline.
It also includes a Web activity for sending a notification to an Azure Logic App.
Repository Contents
- dataset/: SQL, JSON, and Parquet dataset definitions.
- linkedService/: Connection definitions for Azure SQL Database and ADLS Gen2.
- pipeline/: Single-table and multi-table ingestion pipelines.
- factory/: Azure Data Factory definition.
- publish_config.json: Configures the adf_publish publishing branch.
The repository contains configuration definitions, not source data or infrastructure deployment scripts.
Setup
1. Create an Azure Data Factory instance, an Azure SQL Database, and an ADLS Gen2 storage account.
2. Create a storage container named bronze.
3. Connect the repository or a fork to Azure Data Factory through its Git configuration.
4. Update the AzureSqlDatabase1 and datalake linked services with your connection settings and authentication.
5. Prepare your source tables and choose a suitable tracking column for each.
6. Create bronze/<table>_cdc/cdc.json for each table, containing an initial watermark such as {"cdc":"1900-01-01T00:00:00"}.
7. For the single-table pipeline, also create bronze/<table>_cdc/empty.json containing {}.
8. Configure or remove the Logic App notification activity before running the multi-table pipeline.
Choose an initial watermark earlier than the records you want to load and compatible with the source column’s data type. The watermark file is required even when from_date is supplied.
Running the Pipelines
1. Validate the definitions and test both linked-service connections.
2. Run Incremental_Ingestion with parameters for one table.
3. Check the output under bronze/<table>/.
4. Confirm that the watermark file was updated correctly.
5. Add or modify a source record so its tracking value exceeds the saved watermark, then run again.
6. Once individual tables work, run Incremental_Loop with your table configuration.
7. Publish the validated pipelines and add a schedule if needed.
Each execution writes a separate timestamped Parquet batch. Existing files are not merged or updated.
Important Notes
- The source maximum is queried after copying. Concurrent changes could advance the watermark beyond the copied data. For reliable operation, capture an upper watermark before extraction and save it after successfully copying that bounded batch.
- The loop currently writes checkpoint JSON from the full Parquet batch. This should be simplified to a single checkpoint record.
- Reprocessing with from_date can create duplicate records across batches.
- Records at or below the watermark and source deletions are not captured.
- Avoid simultaneous executions that update the same table’s checkpoint.
- Replace and rotate the signed Logic App callback URL in the repository. Correct the alert body’s unquoted run ID and match its field names to your Logic App.
- Source-table scripts, scheduled triggers, and automated tests are not included. Execution must be validated in your Azure environment.
Skills Demonstrated
- Parameterized Azure Data Factory pipelines.
- Incremental extraction using watermarks.
- Dynamic SQL expressions.
- Sequential multi-table processing.
- SQL-to-Parquet ingestion.
- Data lake storage organization.
- Conditional execution and notification integration.
