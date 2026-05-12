# MS SQL Server (MSSQL)

## Supported sync modes

| Sync mode | Supported? |
| :--- | :--- |
| [Full Refresh - Overwrite](https://docs.airbyte.com/platform/using-airbyte/core-concepts/sync-modes/full-refresh-overwrite) | Yes |
| [Full Refresh - Append](https://docs.airbyte.com/platform/using-airbyte/core-concepts/sync-modes/full-refresh-append) | Yes |
| [Full Refresh - Overwrite + Deduped](https://docs.airbyte.com/platform/using-airbyte/core-concepts/sync-modes/full-refresh-overwrite-deduped) | Yes |
| [Incremental Sync - Append](https://docs.airbyte.com/platform/using-airbyte/core-concepts/sync-modes/incremental-append) | Yes |
| [Incremental Sync - Append + Deduped](https://docs.airbyte.com/platform/using-airbyte/core-concepts/sync-modes/incremental-append-deduped) | Yes |

## Output schema

Each stream will be output into its own table in SQL Server. Each table will contain the following metadata columns:

- `_airbyte_raw_id`: A random UUID assigned to each incoming record. The column type in SQL Server is `VARCHAR(MAX)`.
- `_airbyte_extracted_at`: A timestamp for when the event was pulled from the data source. The column type in SQL Server is `BIGINT`.
- `_airbyte_meta`: Additional information about the record. The column type in SQL Server is `TEXT`.
- `_airbyte_generation_id`: Incremented each time a [refresh](https://docs.airbyte.com/operator-guides/refreshes) is executed.  The column type in SQL Server is `TEXT`.

See [here](../../platform/understanding-airbyte/airbyte-metadata-fields) for more information about these fields.

## Getting started

### Setup guide

- MS SQL Server: Azure SQL Database or SQL Server 2016 or later

#### Network access

Make sure your SQL Server database can be accessed by Airbyte. If your database is within a VPC, allow access from the IP address you're using to expose Airbyte.

#### Permissions

Create a dedicated Airbyte user with access to the target database. The user needs permission to create and update schemas, tables, and indexes in any schema Airbyte writes to. At minimum, grant the user:

- `CREATE SCHEMA` permission on the database, unless you create all destination schemas before syncing.
- `CREATE TABLE` permission on the database.
- `ALTER`, `INSERT`, `SELECT`, `UPDATE`, and `DELETE` permissions on each destination schema.

If you use **Bulk** load type with SQL Server, the user also needs `ADMINISTER BULK OPERATIONS` or membership in the `bulkadmin` fixed server role to run `BULK INSERT` against the destination tables. For Azure SQL Database and Azure SQL Managed Instance, grant the user `INSERT` and `ADMINISTER DATABASE BULK OPERATIONS`.

#### Target database

Choose an existing database or create a new database to store synced data from Airbyte.

### Configuration

You'll need the following information to configure the MSSQL destination:

- **Host**
  - The host name of the MSSQL database.
- **Port**
  - The port of the MSSQL database.
- **Database Name**
  - The name of the MSSQL database.
- **Default Schema**
  - The default schema where Airbyte writes tables if the source does not specify a namespace. SQL Server uses `dbo` as the default schema for new database users, but Airbyte defaults this field to `public`. Use `dbo`, `public`, or another schema name that exists or that the Airbyte database user can create.
- **Username**
  - The username which is used to access the database.
- **Password**
  - The password associated with this username.
- **JDBC URL Parameters**
  - Additional properties to pass to the JDBC URL string when connecting to the database formatted as 'key=value' pairs separated by the symbol '&'. (example: key1=value1&key2=value2&key3=value3).
- **SSL Method**
  - The SSL configuration supports three modes: Unencrypted, Encrypted \(trust server certificate\), and Encrypted \(verify certificate\).
    - **Unencrypted**: Do not use SSL encryption on the database connection
    - **Encrypted \(trust server certificate\)**: Use SSL encryption without verifying the server's certificate. This is useful for self-signed certificates in testing scenarios, but should not be used in production.
    - **Encrypted \(verify certificate\)**: Use the server's SSL certificate, after standard certificate verification.
      - **Host Name In Certificate** \(optional\): When using certificate verification, this property can be set to specify an expected name for added security. If this value is present, and the server's certificate's host name does not match it, certificate verification will fail.
- **Load Type**
  - The data load type supports two modes: **Insert** and **Bulk**.
    - **Insert**: Uses SQL `INSERT` statements to load data to the destination table.
    - **Bulk**: Stages CSV files in Azure Blob Storage and uses SQL Server `BULK INSERT` to load data to the destination table. If selected, additional configuration is required:
      - **Azure Blob Storage Account Name** - The name of the [Azure Blob Storage account](https://learn.microsoft.com/azure/storage/blobs/storage-blobs-introduction#storage-accounts).
      - **Azure Blob Storage Container Name** - The name of the [Azure Blob Storage container](https://learn.microsoft.com/azure/storage/blobs/storage-blobs-introduction#containers).
      - **Shared Access Signature** - A [shared access signature (SAS)](https://learn.microsoft.com/azure/storage/common/storage-sas-overview) that grants access to the container. Use either **Shared Access Signature** or **Azure Blob Storage account key**, not both.
      - **Azure Blob Storage account key** - The Azure Blob Storage account key. Use either **Azure Blob Storage account key** or **Shared Access Signature**, not both.
      - **BULK Load Data Source** - The [external data source name configured in SQL Server](https://learn.microsoft.com/sql/t-sql/statements/bulk-insert-transact-sql), which references the Azure Blob container.
      - **Pre-Load Value Validation** - When enabled, Airbyte validates all values before loading them into the destination table. This provides stronger data integrity guarantees but may significantly impact performance.

#### MSSQL with Azure Blob Storage (Bulk upload) setup guide

This section describes how to set up the **Bulk** load type. Airbyte stages data and format files in an Azure Blob Storage container, then runs `BULK INSERT` to load those files into SQL Server.

##### Why use Azure Blob Storage bulk upload?

When handling high data volumes or frequent syncs, row-by-row inserts into MSSQL can become slow and resource-intensive. By staging files in Azure Blob Storage first, Airbyte can:

1. **Aggregate data into bulk files**: Airbyte writes records to Blob Storage in batches, reducing row-by-row insert overhead.
2. **Perform bulk ingestion**: SQL Server uses `BULK INSERT` to load these files directly.

##### Prerequisites

1. **A Microsoft SQL Server instance**
   - Bulk load requires SQL Server 2017 or later, Azure SQL Database, or Azure SQL Managed Instance.
2. **Azure Blob Storage account**
   - A storage account and container, for example `bulk-staging`, where Airbyte stages data files and format files.
3. **Permissions**
   - **Blob Storage**: Permission to create, read, and delete objects in the container.
   - **MSSQL**: Permission to create or modify tables and indexes, and permission to run `BULK INSERT`. For SQL Server, this requires `ADMINISTER BULK OPERATIONS` or membership in the `bulkadmin` fixed server role. For Azure SQL Database and Azure SQL Managed Instance, this requires `INSERT` and `ADMINISTER DATABASE BULK OPERATIONS`.

##### Setup guide

Follow these steps to configure MSSQL with Azure Blob Storage for bulk uploads.

###### 1. Set up Azure Blob Storage

1. **Create a storage account and container**
   - In the Azure Portal, create or reuse a storage account.
   - Within that account, create a container, for example `bulk-staging`, for Airbyte staging files.
2. **Establish access credentials**
   - Use either an Azure Blob Storage account key or a **Shared Access Signature (SAS)** scoped to your container.
   - If you use a SAS token, include **Read**, **Write**, **Delete**, and **List** permissions. Microsoft recommends that SAS tokens used by SQL Server database scoped credentials omit the leading `?`. Use the same token format in the Airbyte **Shared Access Signature** field.

###### 2. Configure MSSQL

See the official [Microsoft documentation](https://learn.microsoft.com/en-us/sql/relational-databases/import-export/examples-of-bulk-access-to-data-in-azure-blob-storage?view=sql-server-ver16) for more details. Below is a simplified overview:

1. **Create a master encryption key if required**
   If your environment requires a master key to store credentials securely, create one:
   ```sql
   CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<your_password>';
   ```

2. **Create a database scoped credential**
   Configure a credential that grants MSSQL access to your Blob Storage using the SAS token. Omit the leading `?` from the SAS token:
   ```sql
   CREATE DATABASE SCOPED CREDENTIAL <credential_name>
   WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
        SECRET = '<your_sas_token_without_leading_question_mark>';
   ```

3. **Create an external data source**
   Point MSSQL to your Blob container using the credential:
   ```sql
   CREATE EXTERNAL DATA SOURCE <data_source_name>
   WITH (
       TYPE = BLOB_STORAGE,
       LOCATION = 'https://<storage_account>.blob.core.windows.net/<container_name>',
       CREDENTIAL = <credential_name>
   );
   ```
   Reference `<data_source_name>` in the connector's **BULK Load Data Source** field. Airbyte uploads files to paths inside the configured container, so the external data source should point to the same container used in the Airbyte configuration.

###### 3. Connector configuration

Supply these values in Airbyte:

1. **MSSQL connection details**
   - The server hostname or IP address, port, database name, username, and password.
2. **Bulk Load Data Source**
   - The external data source you created, for example `<data_source_name>`.
3. **Azure Storage Account and Container**
   - The storage account and container Airbyte uses for staging.
4. **Shared Access Signature** or **Azure Blob Storage account key**
   - The credential Airbyte uses to upload and delete staged files.

See the [Configuration section](#configuration) of this guide for more details on `BULK INSERT` connector configuration.

## Reference

For programmatic configuration with the Airbyte API, PyAirbyte, or Terraform, use these parameter names:

```json
{
  "host": "your-sql-server-host",
  "port": 1433,
  "database": "your_database",
  "schema": "dbo",
  "user": "airbyte_user",
  "password": "your_password",
  "jdbc_url_params": "encrypt=true",
  "ssl_method": {
    "name": "encrypted_trust_server_certificate"
  },
  "load_type": {
    "load_type": "INSERT"
  },
  "tunnel_method": {
    "tunnel_method": "NO_TUNNEL"
  }
}
```

For bulk loading, set `load_type.load_type` to `BULK` and include the Azure Blob Storage fields:

```json
{
  "load_type": {
    "load_type": "BULK",
    "azure_blob_storage_account_name": "mystorageaccount",
    "azure_blob_storage_container_name": "bulk-staging",
    "shared_access_signature": "sv=...",
    "bulk_load_data_source": "MyAzureBlobStorage",
    "bulk_load_validate_values_pre_load": false
  }
}
```

Use `azure_blob_storage_account_key` instead of `shared_access_signature` if you authenticate to Azure Blob Storage with an account key.

## Namespace support

This destination supports [namespaces](https://docs.airbyte.com/platform/using-airbyte/core-concepts/namespaces). The namespace maps to a SQL Server schema.

## Changelog

<details>
  <summary>Expand to review</summary>

| Version    | Date       | Pull Request                                               | Subject                                                                                             |
|:-----------|:-----------|:-----------------------------------------------------------|:----------------------------------------------------------------------------------------------------|
| 2.2.16 | 2026-05-12 | [76946](https://github.com/airbytehq/airbyte/pull/76946) | Upgrade Bulk CDK to 1.0.11 and fix `_ab_cdc_deleted_at` column type so the secondary index on CDC streams can be created. |
| 2.2.15 | 2026-01-26 | [72297](https://github.com/airbytehq/airbyte/pull/72297) | Upgrade CDK to 0.2.0 |
| 2.2.14 | 2025-11-05 | [69130](https://github.com/airbytehq/airbyte/pull/69130) | Upgrade to Bulk CDK 0.1.61. |
| 2.2.13     | 2025-09-24 | [66684](https://github.com/airbytehq/airbyte/pull/66684)   | Pin to CDK artifact                                                                                 |
| 2.2.12     | 2025-06-26 | [62078](https://github.com/airbytehq/airbyte/pull/62078)   | Add SSH tunnel support                                                                              |
| 2.2.11     | 2025-05-30 | [61017](https://github.com/airbytehq/airbyte/pull/61017)   | Integration test fixes                                                                              |
| 2.2.10     | 2025-05-29 | [60897](https://github.com/airbytehq/airbyte/pull/60897)   | Internal fixes                                                                                      |
| 2.2.9      | 2025-05-21 | [60791](https://github.com/airbytehq/airbyte/pull/60791)   | Fix bug in detecting schema change when stream has no columns                                       |
| 2.2.8      | 2025-05-21 | [59735](https://github.com/airbytehq/airbyte/pull/59735)   | Cleanup: Remove unused code                                                                         |
| 2.2.7      | 2025-05-21 | [56444](https://github.com/airbytehq/airbyte/pull/56444)   | CDK: Internal refactor; perf improvements                                                           |
| 2.2.6      | 2025-04-22 | [58146](https://github.com/airbytehq/airbyte/pull/58146)   | Fix numeric bounds-handling                                                                         |
| 2.2.5      | 2025-04-19 | [58140](https://github.com/airbytehq/airbyte/pull/58140)   | Upgrade to latest CDK                                                                               |
| 2.2.4      | 2025-04-17 | [57563](https://github.com/airbytehq/airbyte/pull/57563)   | Improve BULK INSERT documentation.                                                                  |
| 2.2.3      | 2025-04-17 | [58085](https://github.com/airbytehq/airbyte/pull/58085)   | Internal refactoring                                                                                |
| 2.2.2      | 2025-04-09 | [56391](https://github.com/airbytehq/airbyte/pull/56391)   | Add support for Azure blob storage auth via storage account key.                                    |
| 2.2.1      | 2025-03-27 | [56402](https://github.com/airbytehq/airbyte/pull/56402)   | Improve Azure blob storage load logic.                                                              |
| 2.2.0      | 2025-04-02 | [56353](https://github.com/airbytehq/airbyte/pull/56353)   | Bulk Load performance improvements                                                                  |
| 2.1.2      | 2025-03-27 | [56346](https://github.com/airbytehq/airbyte/pull/56346)   | Internal refactor                                                                                   |
| 2.1.1      | 2025-03-24 | [56355](https://github.com/airbytehq/airbyte/pull/56355)   | Upgrade to airbyte/java-connector-base:2.0.1 to be M4 compatible.                                   |
| 2.1.0      | 2025-03-24 | [55849](https://github.com/airbytehq/airbyte/pull/55849)   | Misc. bugfixes in type-handling (esp. in complex types)                                             |
| 2.0.5      | 2025-03-24 | [55904](https://github.com/airbytehq/airbyte/pull/55904)   | Fix handling of invalid schemas (correctly JSON-serialize values)                                   |
| 2.0.4      | 2025-03-20 | [55886](https://github.com/airbytehq/airbyte/pull/55886)   | Internal refactor                                                                                   |
| 2.0.3      | 2025-03-18 | [55811](https://github.com/airbytehq/airbyte/pull/55811)   | CDK: Pass DestinationStream around vs Descriptor                                                    |
| 2.0.2      | 2025-03-12 | [55720](https://github.com/airbytehq/airbyte/pull/55720)   | Restore definition ID                                                                               |
| 2.0.1      | 2025-03-12 | [55718](https://github.com/airbytehq/airbyte/pull/55718)   | Fix breaking change information in metadata.yaml                                                    |
| 2.0.0      | 2025-03-11 | [55684](https://github.com/airbytehq/airbyte/pull/55684)   | Release 2.0.0                                                                                       |
| 2.0.0.rc13 | 2025-03-07 | [55252](https://github.com/airbytehq/airbyte/pull/55252)   | RC13: Bugfix for OOM on Bulk Load                                                                   |
| 2.0.0.rc12 | 2025-03-05 | [54159](https://github.com/airbytehq/airbyte/pull/54159)   | RC12: Support For Bulk Insert Using Azure Blob Storage                                              |
| 2.0.0.rc11 | 2025-03-04 | [55193](https://github.com/airbytehq/airbyte/pull/55193)   | RC11: Increase decimal precision                                                                    |
| 2.0.0.rc10 | 2025-02-24 | [54648](https://github.com/airbytehq/airbyte/pull/54648)   | RC10: Fix index column names with hyphens                                                           |
| 2.0.0.rc9  | 2025-02-21 | [54197](https://github.com/airbytehq/airbyte/pull/54197)   | RC9: Fix index column names with invalid characters                                                 |
| 2.0.0.rc8  | 2025-02-20 | [54186](https://github.com/airbytehq/airbyte/pull/54186)   | RC8: Fix String support                                                                             |
| 2.0.0.rc7  | 2025-02-11 | [53364](https://github.com/airbytehq/airbyte/pull/53364)   | RC7: Revert deletion change                                                                         |
| 2.0.0.rc6  | 2025-02-11 | [53364](https://github.com/airbytehq/airbyte/pull/53364)   | RC6: Break up deletes into loop to reduce locking                                                   |
| 2.0.0.rc5  | 2025-02-07 | [53236](https://github.com/airbytehq/airbyte/pull/53236)   | RC5: Use rowlock hint                                                                               |
| 2.0.0.rc4  | 2025-02-06 | [53192](https://github.com/airbytehq/airbyte/pull/53192)   | RC4: Fix config, timehandling, performance tweak                                                    |
| 2.0.0.rc3  | 2025-02-04 | [53174](https://github.com/airbytehq/airbyte/pull/53174)   | RC3: Fix metadata.yaml for publish                                                                  |
| 2.0.0.rc2  | 2025-02-04 | [52704](https://github.com/airbytehq/airbyte/pull/52704)   | RC2: Performance improvement                                                                        |
| 2.0.0.rc1  | 2025-01-24 | [52096](https://github.com/airbytehq/airbyte/pull/52096)   | Release candidate                                                                                   |
| 1.0.3      | 2025-01-10 | [51497](https://github.com/airbytehq/airbyte/pull/51497)   | Use a non root base image                                                                           |
| 1.0.2      | 2024-12-18 | [49891](https://github.com/airbytehq/airbyte/pull/49891)   | Use a base image: airbyte/java-connector-base:1.0.0                                                 |
| 1.0.1      | 2024-11-04 | [\#48134](https://github.com/airbytehq/airbyte/pull/48134) | Fix supported sync modes (destination-mssql 1.x.y does not support dedup)                           |
| 1.0.0      | 2024-04-11 | [\#36050](https://github.com/airbytehq/airbyte/pull/36050) | Update to Dv2 Table Format and Remove normalization                                                 |
| 0.2.0      | 2023-06-27 | [\#27781](https://github.com/airbytehq/airbyte/pull/27781) | License Update: Elv2                                                                                |
| 0.1.25     | 2023-06-21 | [\#27555](https://github.com/airbytehq/airbyte/pull/27555) | Reduce image size                                                                                   |
| 0.1.24     | 2023-06-05 | [\#27034](https://github.com/airbytehq/airbyte/pull/27034) | Internal code change for future development (install normalization packages inside connector)       |
| 0.1.23     | 2023-04-04 | [\#24604](https://github.com/airbytehq/airbyte/pull/24604) | Support for destination checkpointing                                                               |
| 0.1.22     | 2022-10-21 | [\#18275](https://github.com/airbytehq/airbyte/pull/18275) | Upgrade commons-text for CVE 2022-42889                                                             |
| 0.1.20     | 2022-07-14 | [\#14618](https://github.com/airbytehq/airbyte/pull/14618) | Removed additionalProperties: false from JDBC destination connectors                                |
| 0.1.19     | 2022-05-25 | [\#13054](https://github.com/airbytehq/airbyte/pull/13054) | Destination MSSQL: added custom JDBC parameters support.                                            |
| 0.1.18     | 2022-05-17 | [\#12820](https://github.com/airbytehq/airbyte/pull/12820) | Improved 'check' operation performance                                                              |
| 0.1.17     | 2022-04-05 | [\#11729](https://github.com/airbytehq/airbyte/pull/11729) | Bump mina-sshd from 2.7.0 to 2.8.0                                                                  |
| 0.1.15     | 2022-02-25 | [\#10421](https://github.com/airbytehq/airbyte/pull/10421) | Refactor JDBC parameters handling                                                                   |
| 0.1.14     | 2022-02-14 | [\#10256](https://github.com/airbytehq/airbyte/pull/10256) | Add `-XX:+ExitOnOutOfMemoryError` JVM option                                                        |
| 0.1.13     | 2021-12-28 | [\#9158](https://github.com/airbytehq/airbyte/pull/9158)   | Update connector fields title/description                                                           |
| 0.1.12     | 2021-12-01 | [\#8371](https://github.com/airbytehq/airbyte/pull/8371)   | Fixed incorrect handling "\n" in ssh key                                                            |
| 0.1.11     | 2021-11-08 | [\#7719](https://github.com/airbytehq/airbyte/pull/7719)   | Improve handling of wide rows by buffering records based on their byte size rather than their count |
| 0.1.10     | 2021-10-11 | [\#6877](https://github.com/airbytehq/airbyte/pull/6877)   | Add `normalization` capability, add `append+deduplication` sync mode                                |
| 0.1.9      | 2021-09-29 | [\#5970](https://github.com/airbytehq/airbyte/pull/5970)   | Add support & test cases for MSSQL Destination via SSH tunnels                                      |
| 0.1.8      | 2021-08-07 | [\#5272](https://github.com/airbytehq/airbyte/pull/5272)   | Add batch method to insert records                                                                  |
| 0.1.7      | 2021-07-30 | [\#5125](https://github.com/airbytehq/airbyte/pull/5125)   | Enable `additionalPropertities` in spec.json                                                        |
| 0.1.6      | 2021-06-21 | [\#3555](https://github.com/airbytehq/airbyte/pull/3555)   | Partial Success in BufferedStreamConsumer                                                           |
| 0.1.5      | 2021-07-20 | [\#4874](https://github.com/airbytehq/airbyte/pull/4874)   | declare object types correctly in spec                                                              |
| 0.1.4      | 2021-06-17 | [\#3744](https://github.com/airbytehq/airbyte/pull/3744)   | Fix doc/params in specification file                                                                |
| 0.1.3      | 2021-05-28 | [\#3728](https://github.com/airbytehq/airbyte/pull/3973)   | Change dockerfile entrypoint                                                                        |
| 0.1.2      | 2021-05-13 | [\#3367](https://github.com/airbytehq/airbyte/pull/3671)   | Fix handle symbols unicode                                                                          |
| 0.1.1      | 2021-05-11 | [\#3566](https://github.com/airbytehq/airbyte/pull/3195)   | MS SQL Server Destination Release!                                                                  |

</details>
