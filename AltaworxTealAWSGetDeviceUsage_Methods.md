## AltaworxTealAWSGetDeviceUsage — Methods and Components Reference

This reference explains each method, helper, environment variable, and table involved in the AltaworxTealAWSGetDeviceUsage Lambda. Use it as a quick guide during development and troubleshooting.

### How the Lambda runs (very high level)
- Receives an SQS trigger and inspects message attributes
- If `InitializeProcessing=true`:
  - Seeds the ICCID processing queue table, calculates group count
  - Enqueues processing messages per group
  - Enqueues a notification message
- If `InitializeProcessing=false`:
  - Fetches a batch of ICCIDs for the group
  - Retrieves usage from Teal API
  - Stages results and removes processed ICCIDs
  - Re-enqueues next batch, or finalizes when done

---

## Methods (Initialization Flow)

### StartDailyUsageProcessing
- **Purpose**: Orchestrates the "kickoff" for a provider’s daily usage run.
- **Inputs**: `KeySysLambdaContext context`, `int serviceProviderId`.
- **Actions**:
  - Calls `CallDailyGetUsageSP` to populate `TealDeviceUsageICCIDsToProcess`.
  - Calls `GetGroupCount` to determine the max `GroupNumber` created by the SP.
  - Calls `SendProcessMessagesToQueue` to enqueue a processing message per group.
  - Calls `SendNotificationMessageToQueue` to schedule downstream notification.
- **Output/Side effects**: SQS process messages created; one notification message queued.
- **Errors/Retry**: Logs errors; relies on base retry policies for SQL and SQS.

### CallDailyGetUsageSP
- **Purpose**: Populates the working table of ICCIDs to process for the day.
- **Action**: Executes `usp_Teal_Devices_GetUsageFilter`.
- **Result**: Inserts rows into `TealDeviceUsageICCIDsToProcess` with columns including at least `ICCID`, `ServiceProviderId`, optional `BillingCycleEndDate`, and a computed `GroupNumber` for batching.
- **Notes**: Central place for business rules that pick which devices need usage.

### GetGroupCount
- **Purpose**: Returns the highest group index created during seeding.
- **Query**: `SELECT MAX(GroupNumber) FROM TealDeviceUsageICCIDsToProcess WHERE ServiceProviderId = @ServiceProviderId`.
- **Returns**: Integer (treat `NULL` as no work to do).
- **Usage**: Drives the loop in `SendProcessMessagesToQueue`.

### SendProcessMessagesToQueue
- **Purpose**: Enqueues one SQS message per group to start processing.
- **Inputs**: `serviceProviderId`, `groupCount`.
- **Action**:
  - For each `groupNumber` in `0..groupCount`, send a message to `TealDeviceUsageQueueURL` with attributes: `InitializeProcessing=false`, `ServiceProviderId`, `GroupNumber`.
  - Uses a short delay (e.g., 30 seconds) to stagger starts.
- **Output**: N SQS messages (one per group).

### SendNotificationMessageToQueue
- **Purpose**: Schedules a completion/notification workflow after processing.
- **Action**: Sends an SQS message to `TealDeviceNotificationQueueURL` with attributes such as `RetryCount`, `IntegrationType`, `ServiceProviderId`.
- **Timing**: Uses ~5-minute delay to give processing time to complete.
- **Downstream**: Triggers additional workflows (alerts, syncs, etc.).

---

## Methods (Processing Flow)

### ProcessUsageList
- **Purpose**: Processes a batch of ICCIDs for a given group and provider.
- **Inputs**: `KeySysLambdaContext context`, `GetDeviceUsageSqsValues sqsValues` (contains `ServiceProviderId`, `GroupNumber`).
- **Sequence**:
  1. Read a batch from `TealDeviceUsageICCIDsToProcess` for the given provider and group (ordered by `ICCID`, limited by batch size).
  2. Initialize `DataTable`s via `InitDeviceUsageDataTable` for staging.
  3. Get auth settings via `GetTealAuthenticationInformation`.
  4. Build `TealDeviceDetailService` with retry policy from `GetHttpRetryPolicy`.
  5. Acquire tokens if required (`GetAccessToken`, `GetSessionToken`).
  6. For each ICCID (respecting remaining Lambda time):
     - Call `GetTealUsageAsync` for billing-period/period-range usage.
     - Call `GetTealDailyUsageAsync` for yesterday-only usage.
     - Compute current billing period via `TealCommon.GetBillingPeriod`.
     - Map results to rows using `AddToDataRow`.
     - Delete ICCID from queue table via `RemoveProcessedICCID`.
  7. Bulk insert batches using base `SqlBulkCopy` into staging tables.
  8. If more ICCIDs remain, `SendProcessMessageToQueue` to continue; otherwise `UpdateDeviceUsage` to finalize.
- **Failure handling**: HTTP operations retried (3x exponential backoff). Individual ICCIDs are not retried separately; failures are logged.

### GetTealAuthenticationInformation (from TealCommon)
- **Purpose**: Fetches per-provider Teal credentials and settings from DB.
- **SP**: `usp_Teal_Get_AuthenticationByProviderId`.
- **Returns**: Object with `BaseUrl`, credential fields (`APIKey`/`APISecret` and/or `ClientId`/`ClientSecret`), and billing settings such as `BillPeriodEndDay`/`BillPeriodEndHour`.

### GetAccessToken (via TealDeviceDetailService)
- **Purpose**: Gets an OAuth2 access token when the integration requires OAuth.
- **Call**: POST to token endpoint using Basic auth (`ClientId:ClientSecret`).
- **Returns**: `TealTokenResponse` (access token, expiry).
- **Retries**: 3 attempts with exponential backoff.

### GetSessionToken (via TealDeviceDetailService)
- **Purpose**: Exchanges user credentials plus access token for a session token, when required by the API variant.
- **Call**: POST with username/password and access token.
- **Returns**: `TealLoginResponse` (session token).

### GetTealUsageAsync (via TealDeviceDetailService)
- **Purpose**: Retrieves usage over a billing period window for an ICCID.
- **Request**:
  - Base URL from DB; relative path from env `TealDeviceUsageGetURL` (aka `TealDeviceDataUsageURL`, typically `api/v1/data-consumption/data`).
  - Query params include `offset`, `limit=100`, `dataType=DAILY`, `periodStart`, `periodEnd`.
  - Follows returned `operationResultLink` to obtain results.
- **Returns**: Aggregatable usage history (`DeviceUsageResponseRootObject`).

### GetTealDailyUsageAsync (via TealDeviceDetailService)
- **Purpose**: Retrieves yesterday-only usage for an ICCID.
- **Request**: Same endpoint as above, with `periodStart == periodEnd == yesterday`.
- **Returns**: Daily usage data (if present) in the same format.

---

## Base/Shared Components

### AwsFunctionBase
- **Provides**:
  - `BaseFunctionHandler` for context initialization (logging, clients, configuration).
  - `SqlBulkCopy` for high-throughput inserts to staging tables.
  - `LogInfo`/`LogError` glue and shared telemetry.
  - `CleanUp` to dispose resources.

### TealCommon
- **Static helpers** for Teal integrations:
  - `GetTealAuthenticationInformation` (DB auth lookup).
  - `GetBillingPeriod` (compute current period using provider’s end day/hour via `BillingPeriodHelper`).

### TealDeviceDetailService
- **Role**: Encapsulates all HTTP calls to Teal and related token management.
- **Constructed with**: Auth info, base URL, retry policy, and HTTP client.
- **Exposes**: `GetAccessToken`, `GetSessionToken`, `GetTealUsageAsync`, `GetTealDailyUsageAsync`.
- **Resiliency**: Uses Polly-style retry policy from `GetHttpRetryPolicy`.

---

## Data Movement and Finalization

### UpdateDeviceUsage
- **Purpose**: Finalizes staging data into production tables at the end of processing.
- **SP**: `usp_Teal_Update_Device_Usage` (or `usp_Teal_Update_DeviceUsage_FromStaging` in some environments).
- **Behavior**:
  - Merges staged usage into `TealDeviceUsage` keyed by EID (ICCID joins handled in SQL layer).
  - Inserts history rows and updates last-sync markers.
  - Applies business logic, including zeroing current-period usage where no staged data exists.

### RemoveProcessedICCID
- **Purpose**: Prevents re-processing of a successfully handled ICCID.
- **Action**: `DELETE FROM TealDeviceUsageICCIDsToProcess WHERE ICCID = @iccid AND ServiceProviderId = @serviceProviderId`.

### SendProcessMessageToQueue
- **Purpose**: Continues processing the next batch for the same group.
- **Action**: Sends an SQS message to `TealDeviceUsageQueueURL` with `InitializeProcessing=false`, `ServiceProviderId`, `GroupNumber` and a short delay (e.g., 5 seconds).
- **Paging**: The presence of remaining ICCIDs in the table dictates continued re-enqueueing.

---

## Utilities

### GetMessageQueueValues
- **Purpose**: Parses SQS message attributes into a strongly-typed structure.
- **Attributes expected**: `InitializeProcessing`, `ServiceProviderId`, `GroupNumber`, optional `PageNumber`.
- **Returns**: `GetDeviceUsageSqsValues` used by the handler to choose flow.

### InitDeviceUsageDataTable
- **Purpose**: Creates in-memory `DataTable` schemas that mirror staging tables.
- **Tables**:
  - Billing-cycle staging (e.g., `TealDeviceUsageStaging`): columns for ICCID/EID, bytes/SMS/session counts, service plan, usage window, billing period year/month, CreatedBy/Date, Processed flag.
  - Daily staging (e.g., `TealDeviceDailyUsage`): similar schema plus optional `ServiceProviderId` column.

### AddToDataRow
- **Purpose**: Maps parsed usage objects into `DataRow`s for bulk copy.
- **Behavior**:
  - Handles nulls and type conversions.
  - Aggregates/sums where required (e.g., group by EID before staging).
  - Sets metadata like billing period year/month and audit columns.

### GetHttpRetryPolicy
- **Purpose**: Provides a reusable HTTP resiliency policy.
- **Config**:
  - Attempts: 3
  - Backoff: exponential (≈ 3s, 9s, 27s)
  - Handles: transient HTTP failures/timeouts; optionally adds jitter.

---

## Configuration (Environment Variables)

### TealDeviceNotificationQueueURL
- **What**: SQS queue URL used for downstream notification after processing.
- **Used by**: `SendNotificationMessageToQueue`.

### TealDeviceUsageGetURL
- **What**: Relative path for Teal usage API (often `api/v1/data-consumption/data`).
- **Alias**: May also appear as `TealDeviceDataUsageURL` in some environments.
- **Used by**: `TealDeviceDetailService` (`GetTealUsageAsync`, `GetTealDailyUsageAsync`).

---

## Database Objects

### TealDeviceUsageICCIDsToProcess (table)
- **Role**: Work queue in DB for ICCIDs awaiting processing.
- **Key columns**: `ICCID`, `ServiceProviderId`, `GroupNumber`, optional `BillingCycleEndDate` and additional metadata.
- **Populated by**: `CallDailyGetUsageSP`.
- **Read by**: `ProcessUsageList` (paged batches).
- **Cleaned by**: `RemoveProcessedICCID`.

### TealDeviceUsage (table)
- **Role**: Production table holding the current consolidated usage per device/EID.
- **Populated by**: `UpdateDeviceUsage` stored procedure from staging.
- **Notes**: ICCID/IMSI/MSISDN joins and business logic applied during merge.

---

## Notes and Behaviors
- **Batching**: API page size 100; SQL bulk copy size driven by central `SQLConstant.BatchSize`.
- **Grouping**: Usage grouped by EID for staging; ICCID joins in SQL merge.
- **Retries**: HTTP retried 3x; SQL transient handling via shared retry helper.
- **Timeout safety**: Per-ICCID loop checks remaining Lambda time (>60s) before calls.
- **SQS paging**: No custom visibility timeout logic; short delays (5–30s) used when re-enqueuing.

---

## Quick Reference (SQS attributes)
- **Initialization message**:
  - `InitializeProcessing = true`
  - `ServiceProviderId = <int>`
- **Processing message**:
  - `InitializeProcessing = false`
  - `ServiceProviderId = <int>`
  - `GroupNumber = <int>`
  - Optional: `PageNumber = <int>`

---

Last updated: 2025-09-19