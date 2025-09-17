### AltaworxTealAWSGetDeviceUsage — Q&A and Stored Procedure Mapping

- **Who publishes the initial SQS message for usage processing?**
  - Upstream publisher (not this Lambda). `AltaworxTealAWSGetDeviceUsage` requires an SQS-trigger; if invoked without SQS it logs and returns. It only re-enqueues itself for subsequent pages.

- **What is the batch size/grouping logic for ICCIDs in usage flow?**
  - **API page size**: 100 (`TealHelper.CommonConfig.PAGE_SIZE`).
  - **DB bulk insert batch**: `SQLConstant.BatchSize` for `SqlBulkCopy` (central config).
  - **Grouping**: Usage entries are grouped by `EID` and summed before staging insert. ICCID is not grouped in the Lambda; ICCID joins happen during the SQL MERGE.

- **Confirm ThingSpace API endpoints for RequestTealDeviceUsageAsync**
  - Not ThingSpace. The Lambda uses Teal’s usage API via `TealAPIService.RequestTealDeviceUsageAsync` against the Teal base URL from DB and the relative path from env (typically `api/v1/data-consumption/data`).

- **How are billing periods determined if API data is incomplete?**
  - From DB via `BillingPeriodHelper.GetBillingPeriodForServiceProvider`, using provider `BillPeriodEndDay/Hour`. If the end day for the current month has already passed, it rolls forward one month. It does not depend on API completeness.

- **What happens to failed ICCIDs—are they retried automatically, skipped, or logged?**
  - No per-ICCID handling. HTTP requests are retried at the page/operation level; failures are logged. Individual ICCIDs are neither singled out nor retried separately.

- **Retry config (attempts, delays) for Polly HTTP/SQL retries**
  - **HTTP**: 3 attempts; exponential waits 3s, 9s, 27s; with a fallback that flags error.
  - **SQL**: Wrapped by `RetryPolicyHelper.GetSqlTransientPolicy(...)` (central defaults; attempts/delays defined in shared library).

### Teal Carrier (Device Usage Lambda) — API Details

- **Base URL**
  - From `usp_Teal_Get_AuthenticationByProviderId` per provider (e.g., `https://integrationapi.teal.global/`).

- **Endpoint (relative)**
  - Env var `TealDeviceDataUsageURL` (typically `api/v1/data-consumption/data`).

- **Effective request (conceptual)**
```http
GET {BaseUrl}/api/v1/data-consumption/data
  ?requestId={requestId}
  &offset={offset}
  &limit={pageSize}
  &dataType=DAILY
  &periodStart={yyyy-MM-dd HH:mm:ss}
  &periodEnd={yyyy-MM-dd HH:mm:ss}
```
  - `limit`: 100
  - `offset`: derived from `pageNumber × limit` (computed in service layer)
  - A follow-up GET is performed to the returned `operationResultLink` (HTTPS).

- **Headers**
  - `Accept: application/json`
  - `ApiKey`/`ApiSecret`: from `usp_Teal_Get_AuthenticationByProviderId` (not hard-coded).

### SQS Usage Paging

- **Queue**: `TealDeviceUsageQueueURL` (env).
- **Message attributes**: `ServiceProviderId`, `PageNumber`.
- **Delay/visibility**: Re-enqueue has no explicit `DelaySeconds`. No custom visibility-timeout logic.

### Lambda Flow (concise)

- Read auth via `usp_Teal_Get_AuthenticationByProviderId`.
- Compute billing period via `BillingPeriodHelper`.
- For each page:
  - Call Teal usage API (limit 100) → follow `operationResultLink` → collect entries.
  - Group by `EID`, sum `Usage`, bulk-copy to staging.
  - Also fetch yesterday’s daily usage window and bulk-copy to daily staging if present.
  - Increment `PageNumber` and re-enqueue if more data; otherwise finalize.
- On final page:
  - Execute `usp_Teal_Update_Device_Usage`.
  - Publish cleanup message to `CleanUpQueueURL`.

### Stored Procedures — What and Where

- **`usp_Teal_Get_AuthenticationByProviderId`**
  - **What**: Returns `IntegrationAuthenticationId`, `BaseUrl`, `APIKey`, `APISecret`, `WriteIsEnabled`, `BillPeriodEndDay/Hour`.
  - **Where used**: Builds Teal auth context prior to HTTP calls.

- **`usp_Service_Provider_Get_Bill_Period_Day_And_Hour`**
  - **What**: Returns billing cycle dates and end day/hour for the given month/year.
  - **Where used**: Via `BillingPeriodHelper.GetBillingPeriodForServiceProvider` to compute the current billing period.

- **`usp_Teal_Update_Device_Usage`**
  - **What**: Writes last-sync row; merges staged usage into `TealDeviceUsage` (matching on `EID`, populating ICCID/IMSI/MSISDN/etc. from staging and device tables), inserts history, updates `LastUsageDate`, and zeroes current-month usage where no staged data exists.
  - **Where used**: Called at end of paging to finalize usage data.

### Configuration and Constants

- **Env vars**: `TealDeviceDataUsageURL`, `TealDeviceUsageQueueURL`, `CleanUpQueueURL`.
- **Constants**: `PAGE_SIZE=100`, `RETRY_NUMBER=3`, `SqlBulkCopy.BatchSize=SQLConstant.BatchSize` (central).