### AltaworxTealAWSGetDeviceUsage — Functional Overview

- **Lambda name**: `AltaworxTealAWSGetDeviceUsage`
- **Trigger**: SQS (records consumed via `SQSEvent`)
- **Purpose**: Fetch Teal device usage from Teal APIs and stage it into DB tables; then either re-queue itself for pagination or hand off to a cleanup workflow when complete.

### Business Logic (What it does)
- **Initialization**:
  - Loads environment variables:
    - `TealDeviceDataUsageURL`
    - `TealDeviceUsageQueueURL`
    - `CleanUpQueueURL`
  - Resolves Teal auth via `TealRepository.GetTealAuthenticationInformation`.
  - Determines billing period using `GetBillingPeriod(...)` based on service provider config (end day/hour) and current time.

- **Main loop (pagination)**:
  - Calls Teal Device Usage API for the current billing window (start to end).
  - On success, dereferences operation-result link and materializes `entries` into a list of `TealDeviceUsage`.
  - Groups usage by `EID`, sums `Usage`, and bulk-copies to the staging table.
  - Increments page and continues until a page returns no entries.

- **Daily usage snapshot**:
  - For each pagination cycle, additionally fetches usage for the prior day `[yesterday 00:00, today 00:00)` and bulk-copies grouped results to a daily staging table.

- **Completion / continuation**:
  - If more pages are expected (time remaining and entries found), re-enqueues itself to `TealDeviceUsageQueueURL` with `ServiceProviderId` and `PageNumber` attributes.
  - If last page reached, executes stored procedure `usp_Teal_Update_Device_Usage` to merge staged data into main device usage tables; then enqueues a message to `CleanUpQueueURL` to continue downstream processing (Jasper cleanup for Teal devices).

### Staging Tables (What are staging tables?)
- **Definition**: Temporary intermediate tables used to load raw usage data from Teal before it is merged into the main, normalized reporting tables via a stored procedure. They enable efficient bulk-load and idempotent processing across multiple pages and runs.
- **Used here**:
  - `DatabaseTableNames.TealDeviceUsageStaging`
  - `DatabaseTableNames.TealDeviceUsageDailyStaging`
- **Columns inserted (via DataTable schema built in code)**:
  - `Id` (not populated by code for bulk copy)
  - `EID`
  - `OperationalImsi`
  - `Type`
  - `Period`
  - `Usage` (sum per `EID` for the page/window)
  - `MccMnc`
  - `Pmn`
  - `ServiceProviderId`
  - `CreatedDate` (usageDate passed to the copy function)

### Main Device Usage Update (Merge to primary tables)
- On completion, runs stored procedure: `usp_Teal_Update_Device_Usage`.
- Purpose: Merge records from staging (`TealDeviceUsageStaging`, `TealDeviceUsageDailyStaging`) into the main device usage tables for reporting/analytics. Exact target table names are encapsulated in the stored procedure and not shown in this repo.

### Database Connections and DB Names
- Connection used: `context.CentralDbConnectionString` (SQL Server).
  - Used for: bulk copy into staging; executing `usp_Teal_Update_Device_Usage`; determining billing period.
- Additional DB connections are referenced in shared base code (e.g., `GeneralProviderSettings.JasperDbConnectionString`), but this lambda’s usage operations run against the central database via `CentralDbConnectionString`.
- Exact database name(s) are configured externally and not hardcoded in this repository; they are provided via environment/setting providers tied to the lambda context.

### External Endpoints & Queues
- **Teal API endpoint (data usage)**:
  - Base/path configured by env var: `TealDeviceDataUsageURL`
  - API flow: Request usage -> receive `operationResultLink` -> poll `Operation Result` endpoint to obtain `entries`.
- **SQS re-trigger queue**:
  - Env var: `TealDeviceUsageQueueURL`
  - Message attributes: `ServiceProviderId`, `PageNumber`
- **Cleanup queue**:
  - Env var: `CleanUpQueueURL`
  - Message attributes: `RetryCount`, `IntegrationType` = Teal, `ServiceProviderId`

### What is the endpoint from this lambda?
- This lambda is not invoked via a REST URL. Its "endpoint" is an SQS queue consumer:
  - It processes messages from the queue configured as the event source mapping for `TealDeviceUsageQueueURL`.
  - It also sends messages to `TealDeviceUsageQueueURL` (self-retrigger) and `CleanUpQueueURL` (downstream cleanup), but it does not expose an HTTP endpoint.

### Key Methods (for reference)
- `FunctionHandler(SQSEvent, ILambdaContext)`: entry point; iterates SQS records.
- `ProcessGetDeviceUsage(...)`: orchestrates pagination, staging, daily usage, and continuation/completion.
- `CopyDataToTable(...)`: builds `DataTable` and executes SQL bulk copy to staging.
- `ProcessGetDeviceDataUsage(...)`: calls Teal APIs with retry and materializes results.
- `ProcessFinalProcessGetDeviceUsage(...)`: runs merge SP and enqueues cleanup.
- `GetBillingPeriod(...)`: resolves billing window for the service provider.