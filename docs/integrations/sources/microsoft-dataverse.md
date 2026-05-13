# Microsoft Dataverse

This page contains the setup guide and reference information for the Microsoft Dataverse source connector.

## Sync overview

The Microsoft Dataverse source connector syncs data from [Microsoft Dataverse](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/overview) using the [Dataverse Web API](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/overview). The connector uses Web API version `v9.2`.

The connector discovers Dataverse tables from your environment at setup time. Custom tables and columns can be discovered if the configured application user has permission to read them.

## Prerequisites

To set up this connector, you need:

- A Microsoft Dataverse environment URL, such as `https://<org-id>.crm.dynamics.com`.
- A Microsoft Entra ID tenant associated with your Dataverse environment.
- Administrator access to register an application in Microsoft Entra ID and create an application user in the Dataverse environment.
- A Microsoft Entra application registration configured for server-to-server authentication.
- A Dataverse application user for that application registration, with security roles that grant read access to the tables you want to sync.
- The application \(client\) ID, directory \(tenant\) ID, and client secret value from the application registration.

## Setup guide

### Set up Microsoft Dataverse

The connector authenticates with Microsoft Dataverse using OAuth 2.0 client credentials.

1. Follow Microsoft's [single-tenant server-to-server authentication guide](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/use-single-tenant-server-server-authentication) to create a Microsoft Entra application registration.
2. Record the following values from the app registration:
   - Application \(client\) ID
   - Directory \(tenant\) ID
   - Client secret value
3. In the Power Platform admin center, create an [application user](https://learn.microsoft.com/en-us/power-platform/admin/manage-application-users) for the app registration in your Dataverse environment.
4. Assign security roles to the application user. The roles must grant read access to every table and column you want Airbyte to sync.

### Set up the source connector in Airbyte

<!-- env:cloud -->

**For Airbyte Cloud:**

1. In Airbyte, select **Sources**.
2. Select **New source**.
3. Select **Microsoft Dataverse** from the source type list.
4. For **URL**, enter your Dataverse environment URL. Don't include `/api/data/v9.2`.
5. For **Tenant Id**, enter the directory \(tenant\) ID.
6. For **Client Id**, enter the application \(client\) ID.
7. For **Client Secret**, enter the client secret value.
8. Optionally, set **Max page size**. This controls the `odata.maxpagesize` preference sent to Dataverse. The default is `5000`.
9. Select **Set up source**.

<!-- /env:cloud -->

<!-- env:oss -->

**For Airbyte Open Source:**

1. In Airbyte, select **Sources**.
2. Select **New source**.
3. Select **Microsoft Dataverse** from the source type list.
4. For **URL**, enter your Dataverse environment URL. Don't include `/api/data/v9.2`.
5. For **Tenant Id**, enter the directory \(tenant\) ID.
6. For **Client Id**, enter the application \(client\) ID.
7. For **Client Secret**, enter the client secret value.
8. Optionally, set **Max page size**. This controls the `odata.maxpagesize` preference sent to Dataverse. The default is `5000`.
9. Select **Set up source**.

<!-- /env:oss -->

## Supported sync modes

The Microsoft Dataverse source connector supports the following [sync modes](/platform/using-airbyte/core-concepts/sync-modes/):

| Feature                       | Supported? | Notes                                                                 |
| :---------------------------- | :--------- | :-------------------------------------------------------------------- |
| Full Refresh Sync             | Yes        |                                                                       |
| Incremental Sync              | Yes        | Only for tables with Dataverse change tracking enabled                |
| Change Data Capture \(CDC\)   | Yes        | Uses Dataverse change tracking. Deleted records include only their ID. |
| Replicate Incremental Deletes | Yes        | Only for tables synced incrementally                                  |
| SSL connection                | Yes        |                                                                       |
| Namespaces                    | No         |                                                                       |

## Supported streams

The connector discovers one stream for each Dataverse table the application user can access. The stream name is the Dataverse table logical name.

Incremental sync is available only when the table supports change tracking and change tracking is enabled for that table. During discovery, tables with change tracking enabled are configured with `modifiedon` as the default cursor when that column is available. Other discovered tables are available for full refresh sync.

## Output schema

The connector discovers table and column metadata using the [`EntityDefinitions`](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/webapi/query-metadata-web-api) Web API entity set. It requests only the metadata fields the connector needs with `$select` and `$expand`.

For `DateTime` columns, the connector also reads `DateTimeBehavior` from `DateTimeAttributeMetadata`. Dataverse exposes `DateOnly` columns as dates and `UserLocal` or `TimeZoneIndependent` columns as timestamps.

### Data type mapping

| Dataverse type         | Airbyte type              | Notes                                                      |
| :--------------------- | :------------------------ | :--------------------------------------------------------- |
| `String`               | `string`                  |                                                            |
| `UniqueIdentifier`     | `string`                  |                                                            |
| `DateTime` \(DateOnly\) | `date`                    | Applies when `DateTimeBehavior` is `DateOnly`              |
| `DateTime` \(other\)   | `timestamp with timezone` | Applies to `UserLocal` and `TimeZoneIndependent` behaviors |
| `Integer`              | `integer`                 |                                                            |
| `BigInt`               | `integer`                 |                                                            |
| `Money`                | `number`                  |                                                            |
| `Boolean`              | `boolean`                 |                                                            |
| `Double`               | `number`                  |                                                            |
| `Decimal`              | `number`                  |                                                            |
| `Status`               | `integer`                 |                                                            |
| `State`                | `integer`                 |                                                            |
| `Picklist`             | `integer`                 |                                                            |
| `Lookup`               | `string`                  | Exposed as the lookup ID field                             |
| `Virtual`              | Not synced                | Virtual columns are skipped                                |

Other Dataverse types are discovered as `string`.

## Performance and rate limits

Dataverse applies [service protection API limits](https://learn.microsoft.com/en-us/power-apps/developer/data-platform/api-limits) to API traffic. When Dataverse returns `429 Too Many Requests`, the response includes a `Retry-After` header.

If syncs frequently hit rate limits or time out on large tables, lower **Max page size** to request fewer records per page.

## Reference

This connector uses the Microsoft Dataverse Web API at `/api/data/v9.2`.

| Field | Description |
| :---- | :---------- |
| **URL** | Dataverse environment URL, such as `https://<org-id>.crm.dynamics.com`. Don't include `/api/data/v9.2`. |
| **Tenant Id** | Directory \(tenant\) ID for the Microsoft Entra tenant associated with the Dataverse environment. |
| **Client Id** | Application \(client\) ID for the Microsoft Entra app registration. |
| **Client Secret** | Client secret value for the Microsoft Entra app registration. |
| **Max page size** | Maximum number of records requested per page using the `odata.maxpagesize` preference. Defaults to `5000`. |

## Changelog

<details>
  <summary>Expand to review</summary>

| Version | Date       | Pull Request                                             | Subject                                                                                |
| :------ | :--------- | :------------------------------------------------------- | :------------------------------------------------------------------------------------- |
| 1.0.0 | 2026-05-13 | [77565](https://github.com/airbytehq/airbyte/pull/77565) | Map DateOnly fields to `date` format instead of `date-time`. Add `$select` projection to discovery to reduce metadata payload size. Streams with DateOnly fields require a schema refresh and data reset. |
| 0.1.32 | 2025-05-11 | [60052](https://github.com/airbytehq/airbyte/pull/60052) | Update dependencies |
| 0.1.31 | 2025-05-03 | [59292](https://github.com/airbytehq/airbyte/pull/59292) | Update dependencies |
| 0.1.30 | 2025-04-27 | [58830](https://github.com/airbytehq/airbyte/pull/58830) | Update dependencies |
| 0.1.29 | 2025-04-19 | [57684](https://github.com/airbytehq/airbyte/pull/57684) | Update dependencies |
| 0.1.28 | 2025-04-05 | [57100](https://github.com/airbytehq/airbyte/pull/57100) | Update dependencies |
| 0.1.27 | 2025-03-29 | [56631](https://github.com/airbytehq/airbyte/pull/56631) | Update dependencies |
| 0.1.26 | 2025-03-22 | [56043](https://github.com/airbytehq/airbyte/pull/56043) | Update dependencies |
| 0.1.25 | 2025-03-08 | [55454](https://github.com/airbytehq/airbyte/pull/55454) | Update dependencies |
| 0.1.24 | 2025-03-01 | [54768](https://github.com/airbytehq/airbyte/pull/54768) | Update dependencies |
| 0.1.23 | 2025-02-22 | [54356](https://github.com/airbytehq/airbyte/pull/54356) | Update dependencies |
| 0.1.22 | 2025-02-18 | [46493](https://github.com/airbytehq/airbyte/pull/46493) | Update dependencies |
| 0.1.21 | 2024-09-26 | [45938](https://github.com/airbytehq/airbyte/pull/45938) | Make Dataverse available on Airbyte Cloud |
| 0.1.20 | 2024-09-21 | [45777](https://github.com/airbytehq/airbyte/pull/45777) | Update dependencies |
| 0.1.19 | 2024-09-14 | [45482](https://github.com/airbytehq/airbyte/pull/45482) | Update dependencies |
| 0.1.18 | 2024-09-07 | [45224](https://github.com/airbytehq/airbyte/pull/45224) | Update dependencies |
| 0.1.17 | 2024-08-31 | [44987](https://github.com/airbytehq/airbyte/pull/44987) | Update dependencies |
| 0.1.16 | 2024-08-24 | [44640](https://github.com/airbytehq/airbyte/pull/44640) | Update dependencies |
| 0.1.15 | 2024-08-17 | [44224](https://github.com/airbytehq/airbyte/pull/44224) | Update dependencies |
| 0.1.14 | 2024-08-10 | [43653](https://github.com/airbytehq/airbyte/pull/43653) | Update dependencies |
| 0.1.13 | 2024-08-03 | [43164](https://github.com/airbytehq/airbyte/pull/43164) | Update dependencies |
| 0.1.12 | 2024-07-27 | [42612](https://github.com/airbytehq/airbyte/pull/42612) | Update dependencies |
| 0.1.11 | 2024-07-20 | [42373](https://github.com/airbytehq/airbyte/pull/42373) | Update dependencies |
| 0.1.10 | 2024-07-13 | [41920](https://github.com/airbytehq/airbyte/pull/41920) | Update dependencies |
| 0.1.9 | 2024-07-10 | [41346](https://github.com/airbytehq/airbyte/pull/41346) | Update dependencies |
| 0.1.8 | 2024-07-09 | [41247](https://github.com/airbytehq/airbyte/pull/41247) | Update dependencies |
| 0.1.7 | 2024-07-06 | [40800](https://github.com/airbytehq/airbyte/pull/40800) | Update dependencies |
| 0.1.6 | 2024-06-25 | [40340](https://github.com/airbytehq/airbyte/pull/40340) | Update dependencies |
| 0.1.5 | 2024-06-21 | [39931](https://github.com/airbytehq/airbyte/pull/39931) | Update dependencies |
| 0.1.4 | 2024-06-06 | [39265](https://github.com/airbytehq/airbyte/pull/39265) | [autopull] Upgrade base image to v1.2.2 |
| 0.1.3 | 2024-05-20 | [38397](https://github.com/airbytehq/airbyte/pull/38397) | [autopull] base image + poetry + up_to_date |
| 0.1.2 | 2023-08-24 | [29732](https://github.com/airbytehq/airbyte/pull/29732) | 🐛 Source Microsoft Dataverse: Adjust source_default_cursor when modifiedon not exists |
| 0.1.1 | 2023-03-16 | [22805](https://github.com/airbytehq/airbyte/pull/22805) | Fixed deduped cursor field value update |
| 0.1.0 | 2022-11-14 | [18646](https://github.com/airbytehq/airbyte/pull/18646) | 🎉 New Source: Microsoft Dataverse [python cdk] |

</details>
