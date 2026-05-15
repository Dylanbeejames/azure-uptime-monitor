# Azure Website Uptime Monitor

[LinkedIn — Dylan Bryson](https://www.linkedin.com/in/dylan-bryson-b24952181/)

## Walkthrough

[Loom video coming soon — drop link here once recorded]

---

## Summary

Automated monitoring system that checks a website every 5 minutes for availability, response time, and correct content — and alerts the owner within seconds of any failure via email and SMS.

---

## What This Does

- Checks a target URL every 5 minutes, 24/7
- Tests three things on every check: reachability, response time, and content validity
- Fires an alert within seconds of failure via email and SMS
- Writes every result to Azure Table Storage with timestamp, response time, and pass/fail status
- Presents all data in an Azure Workbook dashboard showing uptime percentage and incident log

---

## Architecture

```
rg-uptime-monitor-dylan
├── Storage Account (stuptimedylan)
│   └── Table: uptimechecks        → every check result written here
├── App Service Plan (Y1 Consumption)
├── Function App (func-uptime-dylan)
│   └── Function: check_website    → timer trigger, runs every 5 minutes
├── Log Analytics Workspace
├── Application Insights           → monitors the Function App itself
├── Monitor Action Group           → email + SMS alert targets
└── Monitor Alert Rule             → fires on SITE DOWN log entries
```

---

## Stack

| Layer | Technology |
|---|---|
| Infrastructure | Terraform (azurerm ~> 3.0) |
| Compute | Azure Functions (Linux, Python 3.10, Consumption plan) |
| Storage | Azure Table Storage |
| Monitoring | Application Insights + Log Analytics |
| Alerting | Azure Monitor Action Group (email + SMS) |
| Language | Python 3.10 |

---

## How It Works

The Function App runs `check_website.py` on a 5-minute CRON timer (`0 */5 * * * *`).

Each execution:
1. Makes an HTTP GET request to the target URL with a 10-second timeout
2. Checks response code — anything other than 200 is a FAIL
3. Checks response time — over 5000ms is flagged as SLOW
4. Checks response content — looks for error indicators in the page body
5. Writes the result to Table Storage with status, response time, and timestamp
6. Logs `SITE DOWN` via `logging.error()` on any failure, which triggers the alert rule

The Monitor alert rule runs a KQL query against Log Analytics every 5 minutes. If it finds any `SITE DOWN` entries, it fires the Action Group — sending email and SMS within seconds.

---

## Deployment

### Prerequisites
- Terraform installed
- Azure CLI installed and authenticated (`az login`)
- Python 3.x installed

### Steps

```bash
# Clone the repo
git clone https://github.com/Dylanbeejames/azure-uptime-monitor.git
cd azure-uptime-monitor

# Deploy infrastructure
terraform init
terraform apply

# Install Python dependencies
cd function_app
pip install -r requirements.txt --target .python_packages/lib/site-packages

# Package and deploy function code
cd ..
mkdir -p deploy_temp/check_website
cp host.json deploy_temp/
cp function_app/requirements.txt deploy_temp/
cp function_app/check_website.py deploy_temp/check_website/
cp function_app/function.json deploy_temp/check_website/
cp -r function_app/.python_packages deploy_temp/
cd deploy_temp && zip -r ../function_deploy.zip . && cd ..

az functionapp deployment source config-zip \
  --resource-group rg-uptime-monitor-<yourname> \
  --name func-uptime-<yourname> \
  --src function_deploy.zip
```

### Verify

```bash
# Check table storage for results (wait 5 minutes after deploy)
az storage entity query \
  --account-name st<yourname> \
  --table-name uptimechecks \
  --account-key "<your-key>" \
  --output table
```

You should see rows with `Status = PASS` appearing every 5 minutes.

---

## Dashboard

An Azure Workbook (`Uptime Monitor Dashboard`) is configured in Azure Monitor with three panels:

- **Check history** — time chart of Total/Passed/Failed checks per 5-minute window
- **Uptime percentage** — stat tile showing overall uptime (currently 100%)
- **Incident log** — grid of any SITE DOWN events (empty when the site is healthy)

---

## Troubleshooting Log

Real issues hit during this build and how they were resolved.

**Subscription quota error on App Service Plan (East US)**
`Operation cannot be completed without additional quota. Current Limit (Total VMs): 0`
The free Azure subscription had a VM quota of 0 in East US. Fixed by switching the deployment region to West US 2 in `terraform.tfvars`.

**Function App showed Runtime version: Error after deploy**
The zip was missing `host.json` at the root level. Azure Functions v2 requires this file to identify the runtime version. Added `host.json` with `"version": "2.0"` and redeployed.

**Function registered but check_website not appearing**
The zip structure had all files at the root. Azure Functions on Linux requires function files inside a named subfolder matching the function name. Restructured the zip so `check_website.py` and `function.json` live inside a `check_website/` directory.

**`WEBSITE_RUN_FROM_PACKAGE` conflict causing 409 on zip deploy**
Terraform sets this app setting for package-based deployment. When deploying via `config-zip` CLI command, the two methods conflict. Removed the setting via `az functionapp config appsettings delete` before redeploying.

**`TableClient` has no attribute `create_table_if_not_exists`**
The method exists on `TableServiceClient`, not `TableClient`. Fixed by calling `table_service.create_table_if_not_exists("uptimechecks")` before getting the table client.

**Table Storage rows not appearing despite successful invocations**
The `PartitionKey` was set to the target URL (`https://example.com`), which contains characters (`/`, `:`) not allowed in Azure Table Storage partition keys. Fixed by hardcoding `PartitionKey` to `"uptime"`.

---

## Cost

Runs on a Consumption plan (Y1). At 12 executions per hour the function costs fractions of a cent per day. Table Storage writes at this volume are negligible. Total estimated cost: under $1/month.

---

## Teardown

```bash
terraform destroy
```
