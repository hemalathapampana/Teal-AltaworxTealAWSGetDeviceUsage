## AltaworxJasperGetDevicesCleanup — Teal Device Cleanup Flow: Functions and Methods

This document explains the responsibilities, inputs/outputs, control flow, side effects, and error handling for the key functions and helpers involved in the Teal device cleanup flow implemented by the Lambda "AltaworxJasperGetDevicesCleanup". It is tailored to the Teal integration specifics provided in the system design.

Contents
- Entry/Initialization
  - Function.InitializeServices
- Event Processing
  - Function.ProcessEventAsync
  - Function.ProcessEventRecordAsync
  - Function.GetMessageQueueValues
  - GetDevicesCleanupSqsValues (data contract)
- Sync and Common Ops
  - Function.SyncDeviceTables
  - Function.SyncTealDevices
  - Function.CountRowsToProcess
  - Function.IsTooManyRetries
  - GenerateTealDeviceSyncSummary
- Email / Reporting
  - Function.SendEmailAsync
  - Function.GetSummaryValues
  - Function.SendEmailSummaryAsync
  - GeneralProviderSettings (config)
  - IntegrationTypeRepository.GetIntegrationTypes
- External Integrations
  - DailySyncAmopApiTrigger.SendNotificationToAmop20


### Function.InitializeServices
Purpose
- Initialize all services and configuration needed for processing Teal cleanup messages.

Responsibilities
- Read environment variables used by Teal processing and summary logging.
- Initialize logging toggles for device sync summary logs.
- Initialize S3 client/wrapper for summary CSV uploads.
- Initialize settings/config repositories (e.g., email recipients, integration type metadata).
- Prepare Base64 or utility services used downstream for attachments/log payloads.

Key Environment
- DEVICE_SYNC_SUMMARY_LOG_S3_BUCKET_NAME (string) — target S3 bucket for CSV logs
- DEVICE_SYNC_SUMMARY_LOG_ENABLE (bool) — enable/disable summary log generation
- DeviceNotificationQueueURL (string) — SQS URL used for requeue on retry
- ConnectionString, BaseMultiTenantConnectionString, PORTAL_CONNECTION_STRING — DB connections
- SnowflakeS3BucketName, SnowflakeS3BucketPath — for historian export (used elsewhere)
- AMOP_20_SYNC_UPDATE_API_URL_KEY — AMOP 2.0 endpoint

Inputs/Outputs
- Input: KeySysLambdaContext (returned by BaseFunctionHandler), IConfiguration/environment
- Output: None (sets up services on the Function instance/context)

Side Effects
- Creates clients for S3/SQS/SES as required.
- Reads and caches configuration values on the Function instance/context.

Error Handling
- Missing required environment variables are logged as errors; execution may abort early.
- Non-fatal optional settings (e.g., log enable) default to safe values.

Pseudo-code
```csharp
void InitializeServices(KeySysLambdaContext ctx) {
  deviceSyncSummaryLogEnable = GetBoolEnv("DEVICE_SYNC_SUMMARY_LOG_ENABLE");
  deviceSyncSummaryLogS3BucketName = GetEnv("DEVICE_SYNC_SUMMARY_LOG_S3_BUCKET_NAME");
  s3 = new S3Wrapper(deviceSyncSummaryLogS3BucketName);
  settingsRepo = new SettingsRepository(PORTAL_CONNECTION_STRING);
  integrationTypeRepo = new IntegrationTypeRepository(PORTAL_CONNECTION_STRING);
  base64Service = new Base64Service();
}
```


### Function.ProcessEventAsync
Purpose
- Entry point for handling a batch of SQS messages (AWS Lambda SQS trigger). Iterates messages and delegates to per-record processing.

Responsibilities
- Iterate `SQSEvent.Records`.
- For each record, call `ProcessEventRecordAsync` with SQL retry policy active.
- Aggregate success/failure metrics; ensure failures are logged with sufficient context.

Inputs/Outputs
- Input: KeySysLambdaContext, SQSEvent
- Output: Task (no return value) — completion indicates the batch is processed

Side Effects
- Logs per-batch and per-message progress.
- May requeue individual items based on retry policy within `ProcessEventRecordAsync` logic.

Error Handling
- Catches unhandled exceptions to avoid Lambda batch failure where possible.
- Uses SQL retry policy (Polly) for transient DB issues.

Pseudo-code
```csharp
async Task ProcessEventAsync(KeySysLambdaContext ctx, SQSEvent evt) {
  foreach (var msg in evt.Records) {
    await ProcessEventRecordAsync(ctx, msg);
  }
}
```


### Function.ProcessEventRecordAsync
Purpose
- Process a single SQS message for Teal cleanup: parse, validate, decide retry/continue, perform sync, summarize, notify.

Responsibilities
- Parse SQS attributes/body using `GetMessageQueueValues` into `GetDevicesCleanupSqsValues`.
- Resolve integration type (must be Teal) using `IntegrationTypeRepository.GetIntegrationTypes`.
- Count rows remaining; check retry ceilings/backoff via `CountRowsToProcess` and `IsTooManyRetries`.
- Execute main Teal sync via `SyncDeviceTables` (invokes `SyncTealDevices` + common sync SP).
- Generate Teal CSV summary to S3 if logging enabled.
- Update communication plans and send “new plans” email if applicable.
- Gather sync summary with `GetSummaryValues` and send email via `SendEmailSummaryAsync`.
- Trigger AMOP 2.0 notification for downstream systems.

Inputs/Outputs
- Input: KeySysLambdaContext, SQS message
- Output: Task (no return value)

Side Effects
- Database writes via stored procedures.
- S3 CSV file uploads for sync summary.
- SES emails to admins.
- Optional SQS requeue when retries are warranted.

Error Handling
- Teal-specific SQL timeout logic (ex.Number == -2) → log + conditional requeue.
- Transient SQL exceptions handled by retry policy.
- Non-critical failures (email/S3) logged and processing continues.

Pseudo-code (high level)
```csharp
async Task ProcessEventRecordAsync(ctx, msg) {
  var sqsValues = GetMessageQueueValues(ctx, msg);
  var integrationTypes = await integrationTypeRepo.GetIntegrationTypes();
  var teal = integrationTypes.First(x => x.Name == "Teal" && x.IsActive);

  var remaining = await CountRowsToProcess(ctx, teal);
  if (IsTooManyRetries(remaining, sqsValues)) {
    await RequeueMessage(sqsValues);
    return;
  }

  await SyncDeviceTables(ctx, sqsValues, sqlRetryPolicy);

  if (deviceSyncSummaryLogEnable) {
    await GenerateTealDeviceSyncSummary(ctx, sqsValues);
  }

  var summary = await GetSummaryValues(ctx, teal, sqsValues.ServiceProviderId);
  await SendEmailSummaryAsync(ctx, teal, summary, shouldGoToHistorian: true, integrationTypes);

  await DailySyncAmopApiTrigger.SendNotificationToAmop20(ctx, ctx.LambdaContext, keyName: "teal_devices", sqsValues.TenantId, null);
}
```


### Function.GetMessageQueueValues
Purpose
- Extract and validate Teal-specific attributes from the SQS message into a strongly-typed contract.

Expected Attributes
- IntegrationType: must be "Teal"
- ServiceProviderId: numeric/string identifier of the Teal service provider
- RetryCount, MaxRetries, DelayBetweenRetries: retry controls
- RemainingRowsToProcess: integer hint; Teal may bypass per-ICCIDs batching

Inputs/Outputs
- Input: KeySysLambdaContext, SQS message
- Output: `GetDevicesCleanupSqsValues`

Validation
- Throws/returns error if IntegrationType != Teal
- Ensures numeric attributes parse; defaults applied for missing optional attributes

Pseudo-code
```csharp
GetDevicesCleanupSqsValues GetMessageQueueValues(ctx, msg) {
  return new GetDevicesCleanupSqsValues {
    IntegrationType = msg.Attributes["IntegrationType"],
    ServiceProviderId = ParseInt(msg.Attributes["ServiceProviderId"]),
    RetryCount = ParseInt(msg.Attributes.GetValueOrDefault("RetryCount", "0")),
    MaxRetries = ParseInt(msg.Attributes.GetValueOrDefault("MaxRetries", "3")),
    DelayBetweenRetries = ParseInt(msg.Attributes.GetValueOrDefault("DelayBetweenRetries", "60")),
    RemainingRowsToProcess = ParseInt(msg.Attributes.GetValueOrDefault("RemainingRowsToProcess", "0"))
  };
}
```


### GetDevicesCleanupSqsValues (data contract)
Fields
- IntegrationType: string — must be "Teal"
- ServiceProviderId: int/string — DB scoping for Teal SP
- RetryCount: int — current attempt
- MaxRetries: int — ceiling for auto-retry
- DelayBetweenRetries: int (seconds) — backoff per attempt
- RemainingRowsToProcess: int — informational
- Optional: TenantId, TenantName — used for AMOP notification if provided

Semantics
- Captures all routing and retry controls from the SQS layer; no per-ICCIDs batching for Teal in this Lambda.


### Function.SyncDeviceTables
Purpose
- Orchestrate database synchronization for Teal by running Teal-specific stored procedure followed by common M2M normalization.

Responsibilities
- Execute `SyncTealDevices` (Teal stored procedure: `usp_Teal_Device_Sync`).
- Execute `usp_DeviceSync_Common` for cross-carrier normalization.
- Apply standard SQL timeout values: Device sync (900s), Common (300s).

Inputs/Outputs
- Input: KeySysLambdaContext, GetDevicesCleanupSqsValues, sqlRetryPolicy
- Output: Task

Error Handling
- Catches `SqlException` timeout (-2) to support targeted requeue.
- Retries transient errors via policy.


### Function.SyncTealDevices
Purpose
- Execute the Teal device sync stored procedure `usp_Teal_Device_Sync` for a given ServiceProviderId.

Stored Procedure
- Name: `usp_Teal_Device_Sync`
- Timeout: 900 seconds (STANDARD_TIMEOUT)
- Parameters: `@ServiceProviderId`

Operations Performed by SP (summary)
- Stage processing from Teal staging tables
- Merge/Update Device table
- Sync usage information
- Update statuses/connectivity
- Validate data integrity
- Cleanup processed staging records
- Handle billing period attach/creation with incomplete data rules

Pseudo-code
```csharp
async Task SyncTealDevices(GetDevicesCleanupSqsValues sqs) {
  await ExecuteStoredProcedureAsync(
    "usp_Teal_Device_Sync",
    new [] { new SqlParameter("@ServiceProviderId", sqs.ServiceProviderId) },
    timeoutSeconds: 900);
}
```


### Function.CountRowsToProcess
Purpose
- Determine count of remaining rows/items relevant to Teal processing; used for retry governance and visibility logging.

Notes
- Teal path typically runs set-based sync (no per-ICCIDs batching here). RemainingRows may reflect staging/backlog estimation.

Outputs
- Integer count of outstanding rows (as known to the system).


### Function.IsTooManyRetries
Purpose
- Decide whether to stop processing and requeue based on current retry count and max configured attempts; may incorporate remaining rows and backoff.

Logic
- If `RetryCount >= MaxRetries` → stop retries for this message.
- Otherwise, apply exponential backoff using `DelayBetweenRetries` and possibly `RemainingRowsToProcess`.

Pseudo-code
```csharp
bool IsTooManyRetries(int remainingRows, GetDevicesCleanupSqsValues v) {
  if (v.RetryCount >= v.MaxRetries) return true;
  return false; // backoff handled by requeue delay logic
}
```


### GenerateTealDeviceSyncSummary
Purpose
- Generate Teal-specific CSV summary of device usage deltas after sync and upload to S3.

Output
- S3 object: `TealDeviceSync_{ServiceProviderId}_{yyyyMMdd_HHmmss}.csv`
- Columns: DeviceId, ICCID, MSISDN, CurrentUsage, PreviousUsage, UsageDelta, LastSyncTimestamp, BillingPeriod

Source Query (representative)
```sql
SELECT d.Id,
       d.ICCID,
       d.MSISDN,
       d.CurrentDataUsage,
       d.PreviousDataUsage,
       d.LastSyncDate,
       d.BillingCycleDate
FROM Device d
WHERE d.ServiceProviderId = @ServiceProviderId
  AND d.LastSyncDate >= @SyncStartTime
ORDER BY d.LastSyncDate DESC;
```

Error Handling
- S3 upload failures are logged and retried per internal retry (if configured); failure does not fail the entire sync.


### Function.SendEmailAsync
Purpose
- Send email notification about newly discovered Teal communication plans that require attention (rate plan mapping, etc.).

Trigger
- After sync, when new values exist in `Device.CommunicationPlan` not present in `JasperCommunicationPlan`.

Content
- Subject: "New Communication Plans Added - Teal"
- Body: List of new plan names and basic counts
- Recipients: From `GeneralProviderSettings`/`ServiceProviderSetting`

Representative Detection Query
```sql
SELECT DISTINCT d.CommunicationPlan
FROM [dbo].[Device] d
LEFT JOIN [dbo].[JasperCommunicationPlan] jcp
  ON jcp.CommunicationPlanName = d.CommunicationPlan
WHERE d.ServiceProviderId = @ServiceProviderId
  AND jcp.CommunicationPlanName IS NULL
  AND d.CommunicationPlan IS NOT NULL;
```

Error Handling
- Email send errors are logged; sync continues.


### Function.GetSummaryValues
Purpose
- Retrieve summary metrics for Teal sync to be used in the email summary report.

Implementation
- Executes `usp_Teal_Devices_Get_Sync_Summary @ServiceProviderId`.

Output Model
```csharp
class TealSyncSummary {
  DateTime? DetailLastSyncDate { get; set; }
  int DetailQueueCount { get; set; }
  int DetailUpdatedCount { get; set; }
  DateTime? UsageLastSyncDate { get; set; }
  int UsageQueueCount { get; set; }
  int UsageUpdatedCount { get; set; }
  int DeviceCount { get; set; }
}
```


### Function.SendEmailSummaryAsync
Purpose
- Send HTML/text email summarizing Teal device sync results.

Inputs
- KeySysLambdaContext
- IntegrationType (Teal) and full integration type list (for subject customization)
- TealSyncSummary values
- shouldGoToHistorian (bool): include historian status details

Content
- Subject: "Teal Device Sync Summary - [ServiceProviderName]"
- HTML Table:
  - Service Provider, Sync Date
  - Total Devices
  - Details Updated, Usage Records Updated
  - Last Detail Sync, Last Usage Sync
- Optional: Snowflake export status

Error Handling
- Log on failure; do not fail overall processing.


### GeneralProviderSettings (config)
Purpose
- Provide per-tenant/service provider settings including email recipients, sender addresses, and notification preferences.

Relevant Settings
- DeviceSyncFromEmail
- DeviceSyncToEmail
- DeviceSyncEmailSubject (override)

Provisioning Example
```sql
INSERT INTO ServiceProviderSetting (ServiceProviderId, SettingName, SettingValue) VALUES
(@TealServiceProviderId, 'DeviceSyncFromEmail', 'noreply@company.com'),
(@TealServiceProviderId, 'DeviceSyncToEmail', 'admin@company.com'),
(@TealServiceProviderId, 'DeviceSyncEmailSubject', 'Teal Device Sync Summary');
```


### IntegrationTypeRepository.GetIntegrationTypes
Purpose
- Load integration types and status flags from DB to validate that Teal integration exists and is active.

Usage
- Subject customization and conditional logic based on integration attributes.

Output
- Collection of IntegrationType { Id, Name, IsActive, ... }

Error Handling
- Log and abort Teal path if Teal type not found/active.


### DailySyncAmopApiTrigger.SendNotificationToAmop20
Purpose
- Inform AMOP 2.0 of Teal sync completion for downstream ingestion/historian processes.

Behavior
- HTTP POST to configured AMOP 2.0 endpoint with JSON payload:
```json
{
  "key_name": "teal_devices",
  "tenant_id": "[TenantId]",
  "tenant_name": "[TenantName]",
  "sync_timestamp": "[ISO8601DateTime]",
  "service_provider_id": "[ServiceProviderId]"
}
```

Notes
- No Polly HTTP retry configured in this project; failures are logged.


## Execution Notes (Teal specifics)
- No per-ICCIDs batching in Lambda; set-based `usp_Teal_Device_Sync` + `usp_DeviceSync_Common`.
- Retry policy: SQL transient retry up to 3 attempts; timeouts logged and may trigger requeue.
- Summary CSV is optional and gated by `DEVICE_SYNC_SUMMARY_LOG_ENABLE` and S3 bucket config.
- Snowflake export path is handled outside this function list; mention included for completeness.


## Glossary of Stored Procedures
- usp_Teal_Device_Sync — Teal detail/usage merge, billing period attach, archive stale devices, write daily usage.
- usp_DeviceSync_Common — Normalize device/status history across carriers.
- usp_Teal_Devices_Get_Sync_Summary — Return last sync dates, queue counts, updated counts, device count.


## Errors and Retries (Teal)
- SQL timeout (-2): logged as "Teal sync timeout"; if under MaxRetries, message is requeued with backoff.
- Data validation errors: logged; processing continues to next items.
- S3/email failures: logged; do not fail the sync.


## Sample Logging Messages
- "[Teal] Processing SQS message for ServiceProviderId={id}, Retry={r}/{max}"
- "[Teal] Executing usp_Teal_Device_Sync (timeout=900s)"
- "[Teal] usp_DeviceSync_Common completed"
- "[Teal] Summary CSV uploaded to s3://{bucket}/{key}"
- "[Teal] Email summary sent to {recipients}"
- "[Teal] AMOP 2.0 notified for tenant={tenantId}"


## Security and Compliance
- Ensure least-privilege IAM policies for S3 (write to summary bucket) and SES send-email.
- Protect DB connection strings via AWS Secrets Manager/SSM where possible.
- Validate and sanitize message attributes; treat all external inputs as untrusted.


---

This document is specific to the Teal flow and the Lambda consumer "AltaworxJasperGetDevicesCleanup" based on the provided architecture and behavior. It is intended as an operational and developer reference for maintenance, troubleshooting, and onboarding.