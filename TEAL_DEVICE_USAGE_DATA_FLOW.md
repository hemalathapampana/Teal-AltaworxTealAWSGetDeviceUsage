### Teal device usage → Teal device tables: end‑to‑end data flow (step‑by‑step)

This document explains how device usage data is fetched from Teal, staged, and merged into database tables (including the update/merge procedure). It references the code in this repo and the database objects it calls.

### Actors and key components
- **AWS Lambda function**: `AltaworxTealAWSGetDeviceUsage.Function` (SQS-triggered)
- **Teal API client**: `TealAPIService`
- **Retry policy**: `TealHelper.BuildRetryPolicy` (Polly)
- **Repository**: `TealRepository` to get Teal API credentials from DB
- **Billing period**: `BillingPeriodHelper`
- **Staging tables**: `DatabaseTableNames.TealDeviceUsageStaging`, `DatabaseTableNames.TealDeviceUsageDailyStaging`
- **Merge stored procedure**: `usp_Teal_Update_Device_Usage`
- **Queues**: `TealDeviceUsageQueueURL` (continue pagination), `CleanUpQueueURL` (post-processing)

### Environment variables and constants
```19:37:/workspace/TealHelper.cs
public const string TEAL_DEVICES_GET_URL = "TealDevicesGetURL";
public const string TEAL_RATE_PLANS_URL = "TealRatePlansGetURL";
public const string TEAL_ASSIGN_RATE_PLAN_URL = "TealAssignRatePlanURL";
public const string TEAL_DESTINATION_QUEUE_GET_DEVICES_URL = "TealDestinationQueueGetDevicesURL";
public const string TEAL_DESTINATION_QUEUE_GET_RATE_PLANS_URL = "TealDestinationQueueGetRatePlansURL";
public const string TEAL_DEVICE_USAGE_QUEUE_URL = "TealDeviceUsageQueueURL";
public const string TEAl_DEVICE_API = "api/v1/esims";
public const string TEAl_DEVICE_DATA_USAGE_API = "api/v1/data-consumption/data";
public const string TEAl_DEVICE_SMS_USAGE_API = "api/v1/data-consumption/sms";
public const string TEAl_CARRIER_RATE_PLAN_API = "api/v1/plans";
public const string TEAL_DEVICES_DATA_USAGE_URL = "TealDeviceDataUsageURL";
public const string TEAL_DEVICES_SMS_DATA_USAGE_URL = "TealDeviceSMSDataUsageURL";
public const string CLEAN_UP_QUEUE_URL = "CleanUpQueueURL";
public const string TEAL_GET_DEVICE = "GetDevice";
public const string TEAL_GET_DEVICE_USAGE = "GetUsage";
public const string TEAL_GET_RATE_PLAN = "GetRatePlan";
public const string TEAL_ASSIGN_RATE_PLAN = "AssignRatePlan";
public const string TEAL_GET_DEVICE_SMS_USAGE = "GetSMSUsage";
public const string TEAL_UPDATE_DEVICE_STATUS = "UpdateStatus";
```

### Step 1 — Trigger and input
- The Lambda is triggered by an SQS message carrying `ServiceProviderId` and `PageNumber`.
- If there are multiple pages of usage, the function re-queues itself with the next `PageNumber`.

```316:341:/workspace/AltaworxTealAWSGetDeviceUsage.cs
var request = new SendMessageRequest
{
    MessageAttributes = new Dictionary<string, MessageAttributeValue>
    {
        { "ServiceProviderId", new MessageAttributeValue { DataType = "String", StringValue = sqsValues.ServiceProviderId.ToString() } },
        { "PageNumber", new MessageAttributeValue { DataType = "String", StringValue = sqsValues.PageNumber.ToString() } }
    },
    MessageBody = requestMsgBody,
    QueueUrl = deviceUsageQueueURL
};
```

### Step 2 — Get Teal credentials from DB
- The function loads Teal API credentials and billing settings for the `ServiceProviderId` via a stored procedure.

```24:45:/workspace/TealRepository.cs
using (var sqlCommand = new SqlCommand(Amop.Core.Constants.SQLConstant.StoredProcedureName.usp_Teal_Get_AuthenticationByProviderId, sqlConnection))
{
    sqlCommand.CommandType = CommandType.StoredProcedure;
    sqlCommand.Parameters.AddWithValue("@providerId", serviceProviderId);
    ... // maps API key/secret and billing end day/hour
}
```

### Step 3 — Determine billing period
- Uses a stored procedure to determine bill period boundaries (day/hour) for the provider.

```19:23:/workspace/BillingPeriodHelper.cs
cmd.CommandType = CommandType.StoredProcedure;
cmd.CommandText = "dbo.usp_Service_Provider_Get_Bill_Period_Day_And_Hour";
cmd.Parameters.AddWithValue("@ServiceProviderId", serviceProviderId);
cmd.Parameters.AddWithValue("@BillMonth", billingPeriodMonth);
cmd.Parameters.AddWithValue("@BillYear", billingPeriodYear);
```

### Step 4 — Request usage from Teal (paginated)
- For each page:
  - Calls Teal usage API with dataType `DAILY` and `periodStart`/`periodEnd` for the computed billing period.
  - Teal returns an operation-result link; the function then fetches the actual results from that link.
  - A retry policy wraps both calls.

```185:201:/workspace/AltaworxTealAWSGetDeviceUsage.cs
var tealDeviceUsageResult = await TealHelper.BuildRetryPolicy(string.Empty, TealHelper.CommonString.TEAL_GET_DEVICE_USAGE, context.logger)
    .ExecuteAsync(async () => await tealAPIService.RequestTealDeviceUsageAsync(
        deviceUsagePath, sqsValues.PageNumber, TealHelper.CommonString.TEAL_DATA_TYPE_DAILY,
        startDate, endDate, TealHelper.CommonConfig.PAGE_SIZE, context.logger));
...
var operationResultLink = tealDeviceUsageApiResponse.Link.OperationResultLink.Href;
operationResultLink = operationResultLink.Replace(TealHelper.CommonString.HTTP, TealHelper.CommonString.HTTPS);
var tealOperationResult = await TealHelper.BuildRetryPolicy(operationResultLink, TealHelper.CommonString.OPERATION_RESULT, context.logger)
    .ExecuteAsync(async () => await tealAPIService.GetTealOperationResultAsync(operationResultLink, context.logger));
```

```207:208:/workspace/AltaworxTealAWSGetDeviceUsage.cs
var tealDeviceResponseRootObject = JsonConvert.DeserializeObject<TealDeviceUsageResponseRootObject>(tealOperationResult.ResponseObject);
tealDeviceUsages = tealDeviceResponseRootObject.Entries;
```

### Step 5 — Group and stage usage
- The function groups usage by `EID` and sums `Usage` per `EID` for the page.
- It then bulk-copies rows into a staging table.

```164:178:/workspace/AltaworxTealAWSGetDeviceUsage.cs
var tealDeviceUsageGroup = tealDeviceUsages
    .GroupBy(x => x.EID,
    (key, group) => new TealDeviceTotalUsage() { EID = key, Usage = group.Sum(x => x.Usage), DeviceUsage = group.FirstOrDefault() })
    .ToList();
var tealDeviceDataUsageTable = BuildTealDeviceDataUsageTable();
...
SqlBulkCopy(context, context.CentralDbConnectionString, tealDeviceDataUsageTable, tableName);
```

- Columns staged per row:

```220:230:/workspace/AltaworxTealAWSGetDeviceUsage.cs
tealDeviceUsageTable.Columns.Add(CommonColumnNames.Id);
tealDeviceUsageTable.Columns.Add(CommonColumnNames.EID);
tealDeviceUsageTable.Columns.Add(CommonColumnNames.OperationalImsi);
tealDeviceUsageTable.Columns.Add(CommonColumnNames.Type);
tealDeviceUsageTable.Columns.Add(CommonColumnNames.Period);
tealDeviceUsageTable.Columns.Add(CommonColumnNames.Usage);
tealDeviceUsageTable.Columns.Add(CommonColumnNames.MccMnc);
tealDeviceUsageTable.Columns.Add(CommonColumnNames.Pmn);
tealDeviceUsageTable.Columns.Add(CommonColumnNames.ServiceProviderId);
tealDeviceUsageTable.Columns.Add(CommonColumnNames.CreatedDate);
```

- Field mapping from API entry → staging row:

```240:248:/workspace/AltaworxTealAWSGetDeviceUsage.cs
tealDeviceUsageRow[CommonColumnNames.EID] = device.EID;
tealDeviceUsageRow[CommonColumnNames.OperationalImsi] = device.OperationalImsi;
tealDeviceUsageRow[CommonColumnNames.Type] = device.Type;
tealDeviceUsageRow[CommonColumnNames.Period] = device.Period;
tealDeviceUsageRow[CommonColumnNames.Usage] = deviceUsageTotal.Usage;
tealDeviceUsageRow[CommonColumnNames.MccMnc] = device.MccMnc;
tealDeviceUsageRow[CommonColumnNames.Pmn] = device.PMN;
```

- Staging targets used by the function:

```123:127:/workspace/AltaworxTealAWSGetDeviceUsage.cs
if (deviceDataUsage.Count > 0)
{
    CopyDataToTable(context, deviceDataUsage, sqsValues, DatabaseTableNames.TealDeviceUsageStaging, DateTime.Now);
    sqsValues.PageNumber = sqsValues.PageNumber + 1;
}
```

```144:147:/workspace/AltaworxTealAWSGetDeviceUsage.cs
if (deviceDailyUsage.Count > 0)
{
    CopyDataToTable(context, deviceDataUsage, sqsValues, DatabaseTableNames.TealDeviceUsageDailyStaging, yesterdayMidNight);
}
```

### Step 6 — Pagination
- If a page returned rows, the function increments the page and continues. If not, it marks `isLastPage = true`.
- If time is nearly exhausted, the function re-queues itself to continue on the next invocation.

### Step 7 — Merge/update into final tables
- Once the last page is processed, the function executes the stored procedure `usp_Teal_Update_Device_Usage` (passing `@ServiceProviderId`).
- This procedure is responsible for merging rows from the staging tables into the production usage tables and updating the corresponding device-level aggregates/links (e.g., `TealDevice*` tables) as defined in the database.

```275:289:/workspace/AltaworxTealAWSGetDeviceUsage.cs
using (var command = new SqlCommand(AMOPSQLConstants.StoredProcedureName.usp_Teal_Update_Device_Usage, connection))
{
    command.CommandType = CommandType.StoredProcedure;
    connection.Open();
    command.Parameters.AddWithValue("@ServiceProviderId", serviceProviderId);
    command.CommandTimeout = AMOPSQLConstants.TimeoutSeconds;

    var affectedRows = command.ExecuteNonQuery();
    if (affectedRows > 0)
    {
        LogInfo(context, LogTypeConstant.Info, LogCommonStrings.MERGED_DEVICE_USAGE_SUCCESSFULLY);
    }
}
```

### Step 8 — Post‑processing cleanup
- After a successful final merge, the function sends a message to `CleanUpQueueURL` to kick off downstream cleanup/sync (e.g., consolidating device info and usage for Teal devices).

```253:258:/workspace/AltaworxTealAWSGetDeviceUsage.cs
UpdateTealDeviceUsageWithPolicy(context, sqsValues.ServiceProviderId, context.CentralDbConnectionString, tealAuthentication, context.logger);
SendMessageToJasperDeviceCleanUpQueue(context, _cleanUpQueueURL, sqsValues.ServiceProviderId);
```

### What specifically ties TealDeviceUsage and TealDevice?
- The link is made in the database layer by `usp_Teal_Update_Device_Usage` when it merges from staging into the production schemas and updates device-level data. The application code intentionally delegates this to the stored procedure so schema details and device linkage stay centralized in SQL.

### Daily vs. billing‑period data
- The function stages two kinds of usage snapshots:
  - Aggregated across the current billing period into `TealDeviceUsageStaging`
  - Previous-day usage into `TealDeviceUsageDailyStaging`
- Both flows are finalized by the same stored procedure (`usp_Teal_Update_Device_Usage`).

### Errors, retries, and resilience
- API calls are wrapped by a retry policy.
- Database operations (the final merge) execute under a SQL transient retry policy (see `UpdateTealDeviceUsageWithPolicy`).
- If time is running out, the function re-queues itself to continue; no partial page is lost because data is written in idempotent batches to staging.

### Quick summary
1. SQS triggers the Lambda with `ServiceProviderId` and page.
2. Lambda loads Teal credentials and billing period from SQL.
3. Lambda requests usage from Teal (DAILY) and fetches operation result.
4. Usage rows are grouped by `EID`, mapped to a schema, and bulk-copied to staging.
5. If more pages exist, the function re-queues itself; otherwise it calls `usp_Teal_Update_Device_Usage` to merge staging data into final tables and update `TealDevice` data.
6. Finally, it sends a message to the cleanup queue for downstream sync.
