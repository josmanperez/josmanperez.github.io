---
title: Automating MongoDB Atlas Trigger Log Downloads Beyond the GUI and CLI Limitations
date: 2025-01-24 09:00:00 +/-0000
author: <josman>
categories: [MongoDB Atlas]
tags: [mongodb, atlas, triggers]
description: This tool is a Python script designed to fetch logs from a MongoDB Atlas App Services application using pagination. It supports optional date range filtering and provides a way to authenticate using MongoDB Atlas API keys.
toc: true
image: 
  path: ../assets/img/blog/automating-logs-download.png
  alt: Introductory image
comments: true
---

I recently encountered a scenario where I needed to download extensive logs from MongoDB Atlas Triggers locally. Currently, there are three ways to download logs from Atlas:

1. [Using the GUI](https://www.mongodb.com/docs/atlas/atlas-ui/triggers/logs/)
2. [Using the CLI](https://www.mongodb.com/docs/atlas/app-services/cli/appservices-logs-list/)
3. [Using the App Services Admin API](https://www.mongodb.com/docs/atlas/app-services/admin/api/v3/#tag/logs/Pagination-with-the-Logging-API)

However, the GUI and CLI options have limitations regarding the volume of logs that can be downloaded, particularly a cap of 10,000 logs.

## Limitations of the GUI and CLI for Log Downloads

### Using the GUI

With the GUI, users can filter logs by date, type, user ID, or request ID, but the limit is set to 10,000 logs when it comes to downloading.

### Using the CLI

With the CLI, we can run a command like:

```shell
appservices logs list - project 5e208aa2d5ec1375ecd5*** - app triggers_realmapp-**** - type=trigger - start="2024–10–15T00:00:00.000+0000" -o log.logs
```

> However, this also has the same download limit of 10,000 logs.
{: .prompt-info }

## A Solution: App Services Admin API with Pagination

To overcome these download limitations, the App Services Admin API provides a way to access logs with pagination. By implementing pagination, users can fetch logs beyond the default 10K limit.

Detailed instructions on using pagination with the API can be found in MongoDB’s documentation: [Get Paginated Logs](https://www.mongodb.com/docs/atlas/app-services/activity/view-logs/#get-paginated-logs).

## Proposed Solution: An Automated Script for Large Log Downloads

To streamline this, I developed a script that automatically fetches logs using pagination. This script is available in a public repository here: [Atlas App Logs Aggregator](https://github.com/mongodb-labs/atlas-app-logs-aggregator).

### Key Features of the Script

* **Automated Log Fetching**: Fetches logs from MongoDB Atlas App Services with support for large log sets through pagination.
* **Flexible Date Range Filtering**: Allows optional date filtering using `start_date` and `end_date` parameters.
* **ISO 8601 Validation**: Validate dates to ensure they follow the [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) format.
* **Secure Authentication**: Supports authentication using MongoDB Atlas public and private API keys.
* Optional `--user_id` for user ID filtering logs.
* Optional `--co_id` for correlation ID filtering logs.
* Fetch only error logs using the `--errors_only` option.
* Filter logs by `key-value` pairs using the `--filter` option.

The script only uses the GET endpoint and aggregates logs into a file without modifying any data.

## Requirements

* Python 3.6 or higher.
* `requirements.txt` library dependencies.

## Installation

### Create a virtual environment

```bash
python3 -m venv venv
source venv/bin/activate  # On Windows use `venv\Scripts\activate`
```

### Install dependencies

```bash
pip install -r requirements.txt
```

## Usage

### Command-Line Arguments

* `project_id` (**required**): The Atlas Project ID (hexadecimal string).
app_id (**required**): The App ID (string).
* `public_api_key` (**required**): The Atlas Public API Key (string).
* `private_api_key` (**required**): The Atlas Private API Key (string with hyphens).
* `--start_date` (optional): Start Date in ISO 8601 format (YYYY-MM-DDTHH:MM:SS.MMMZ).
* `--end_date` (optional): End Date in ISO 8601 format (YYYY-MM-DDTHH:MM:SS.MMMZ).
* `--type` (optional): Comma-separated list of supported log types. Currently, the types available are: `TRIGGER_FAILURE, TRIGGER_ERROR_HANDLER, DB_TRIGGER, AUTH_TRIGGER, SCHEDULED_TRIGGER, FUNCTION, SERVICE_FUNCTION, STREAM_FUNCTION, SERVICE_STREAM_FUNCTION, AUTH, WEBHOOK, ENDPOINT, PUSH, API, API_KEY, GRAPHQL, SYNC_CONNECTION_START, SYNC_CONNECTION_END, SYNC_SESSION_START, SYNC_SESSION_END, SYNC_CLIENT_WRITE, SYNC_ERROR, SYNC_OTHER, SCHEMA_ADDITIVE_CHANGE, SCHEMA_GENERATION, SCHEMA_VALIDATION, LOG_FORWARDER`
* `--user_id` (optional): Return only log messages associated with the given user_id.
* `--co_id` (optional): Return only log messages associated with the given request Correlation ID.
* `--filter` (optional): Filter logs by key-value pairs (e.g., `--filter event_subscription_name=<trigger_name>,function_name=<function_name>`).
* `--errors_only` (optional): Return only error log messages.
* `--verbose` (optional): Enable verbose logging information.

### Example

```bash
python main.py <project_id> <app_id> <public_api_key> <private_api_key> --start_date 2024-10-05T14:30:00.000Z --end_date 2024-10-06T14:30:00.000Z --type TRIGGER_FAILURE,SCHEMA_GENERATION
```

With optional parameters

```bash
python main.py <project_id> <app_id> <public_api_key> <private_api_key> --start_date 2024-10-05T14:30:00.000Z --type TRIGGER_FAILURE,SCHEMA_GENERATION --user_id 671d2e2010733ecbaa2bab8f --filter event_subscription_name=getUnpausedClustersMetrics
```

If `start_date` and `end_date` are not provided, the script will default `start_date` to the last 24 hours from the current time.

## Filtering Logs

The `--filter` option allows you to filter logs by key-value pairs. This option accepts multiple key-value pairs separated by spaces. Each key-value pair should be in the format key=value.

The `key-value` pair must be the values returned by the endpoint. This way it will use them to filter and only keep those that match. For example, for a `"type": "SCHEDULED_TRIGGER"`, the response key-values will be similar to:

```json
{
  "_id": "671d2e2010733ecbaa2bab8f",
  "co_id": "671d2e2010733ecbaa2bab8d",
  "type": "SCHEDULED_TRIGGER",
  "domain_id": "65b0fc719629ac8e4d8e8774",
  "app_id": "65b0fc719629ac8e4d8e8773",
  "group_id": "658d46ca7605526eb45222a4",
  "request_url": "",
  "request_method": "",
  "started": "2024-10-26T18:00:00.041Z",
  "completed": "2024-10-26T18:00:04.124Z",
  "function_id": "65f31f9f3bfc77348cb1e2e7",
  "function_name": "getOrgClustersProjects",
  "error": "FunctionError: Cannot access member 'db' of undefined",
  "event_subscription_id": "65f335c53d26a2b1ba5d7ba2",
  "event_subscription_name": "getUnpausedClustersMetrics",
  "messages": [
      "reading projects for page: 1",
      "hay m\u00e1s p\u00e1ginas",
      "reading projects for page: 2",
      "fin"
  ],
  "mem_time_usage": 4081000000
}
```

We can use any of this in the `--filter` option (e.g., `--filter event_subscription_name=getUnpausedClustersMetrics`)

## Logging

The script supports logging to both the console and a log file. By default, log files are stored in the logs folder. The log file name includes a timestamp to ensure uniqueness for each run.

`--verbose`: When this flag is used, the log level is set to `DEBUG`, providing detailed logging information. Without this flag, the log level is set to `INFO`.

### Log File Location

Log files are stored in the logs folder. Each log file is named with a timestamp to ensure that logs from different runs do not overwrite each other.

### Example Log File Name

```bash
logs/app_20241005_143000.log
```

## Benefits

* **Automated Log Retrieval**: Easily fetch logs from MongoDB Atlas App Services without manual intervention.
* **Date Range Filtering**: Filter logs by date range to focus on specific periods.
* **Pagination Support**: Handle large sets of logs efficiently using pagination.
* **Validation**: Ensure date inputs are in the correct format to avoid errors.

## DISCLAIMER

Please note: This repo is released for use "AS IS" without any warranties of any kind, including, but not limited to their installation, use, or performance. We disclaim any and all warranties, either express or implied, including but not limited to any warranty of noninfringement, merchantability, and/ or fitness for a particular purpose. We do not warrant that the technology will meet your requirements, that the operation thereof will be uninterrupted or error-free, or that any errors will be corrected.

Any use of these scripts and tools is at your own risk. There is no guarantee that they have been through thorough testing in a comparable environment and we are not responsible for any damage or data loss incurred with their use.

You are responsible for reviewing and testing any scripts you run thoroughly before use in any non-testing environment.