### Teal Device Sync Flow — Q&A and Stored Procedure Mapping (AltaworxTealAWSGetDevices)

- **Who publishes the first SQS message to `TealDestinationQueueGetDevicesURL`?**
  - `AltaworxTealAWSGetDevices` self-publishes the first page. The flow is kicked off by AWS EventBridge Scheduler (no SQS trigger for the first run). The destination queue URL is taken from the environment variable `TealDestinationQueueGetDevicesURL`.
  - Messages are sent with an explicit delay of 30 seconds (`DELAY_SQS_MESSAGE_IN_SECONDS = 30`).
  - Page-level failure threshold is 5 (`TEAL_SYNC_FAIL_ACCEPTABLE_LIMIT = 5`). On reaching this threshold, the Lambda stops paging for that provider, merges from staging, triggers usage fetch, and proceeds.

- **Environment variables**
  - `TealDevicesGetURL`: Relative endpoint segment for device list (e.g., `api/v1/esims`).
  - `TealDestinationQueueGetDevicesURL`: SQS queue URL for device list processing (used for self-enqueue).
  - `TealDeviceUsageQueueURL`: SQS queue URL for device usage processing (chained after devices phase completes).

- **Operational constants (from Lambda)**
  - `DELAY_SQS_MESSAGE_IN_SECONDS = 30`
  - `TEAL_SYNC_FAIL_ACCEPTABLE_LIMIT = 5`

- **Triggering note**
  - The initial Lambda invocation is triggered by AWS EventBridge. Subsequent paging is driven by SQS messages published to the queue obtained from `TealDestinationQueueGetDevicesURL`.