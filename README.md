# Cloud Resume API + Real‑Time Monitoring (Azure + Terraform)

This project extends my Cloud Resume API with real‑time monitoring using Azure Monitor, Application Insights, and Terraform.  
Every time someone accesses the `/api/getresume` endpoint, Azure logs the request, evaluates a KQL alert rule, and sends an email notification via an Action Group.

This demonstrates:
- Cloud observability
- Infrastructure as Code
- Event‑driven automation
- Debugging Terraform/Azure drift
- Practical monitoring design

---

# 🏗️ Architecture Overview

```
User
  ↓
Azure Function App (Get Resume API)
  ↓ logs
Application Insights
  ↓ KQL query (scheduled)
Azure Monitor Alert Rule
  ↓
Action Group (Email)
  ↓
Inbox Notification
```

---

# 🧩 Components

### Azure Function App
- Hosts the `/api/getresume` endpoint  
- Logs all HTTP requests to Application Insights  

### Application Insights
- Stores request telemetry  
- Provides KQL query engine  
- Powers the alert rule  

### Azure Monitor Alert Rule
- Runs a scheduled KQL query  
- Detects resume API access  
- Triggers the action group  

### Action Group
- Sends email notifications  
- Uses a placeholder email in this README  

### Terraform
Manages:
- Monitoring resource group  
- Action group  
- Alert rule  
- App Insights data source  
- State imports to fix drift  

---

# 📜 KQL Query Used for Monitoring

```kql
requests
| where url contains "/api/getresume"
| summarize Count = count() by bin(timestamp, 5m)
| where Count > 0
```

This query:
- Filters resume API calls  
- Counts them in 5‑minute windows  
- Fires the alert if at least one request is detected  

---

# 🛠️ Terraform Configuration

### Resource Group
```hcl
resource "azurerm_resource_group" "monitoring" {
  name     = "rg-resume-monitoring"
  location = "ukwest"
}
```

### Action Group
```hcl
resource "azurerm_monitor_action_group" "resume_alerts" {
  name                = "ag-resume-alerts"
  resource_group_name = azurerm_resource_group.monitoring.name
  short_name          = "resumeag"

  email_receiver {
    name          = "alert-email"
    email_address = "<your-email-here>"
  }
}
```

### Application Insights Data Source
```hcl
data "azurerm_application_insights" "ai" {
  name                = "function-resume02"
  resource_group_name = "function-resume02_group"
}
```

### Alert Rule
```hcl
resource "azurerm_monitor_scheduled_query_rules_alert_v2" "resume_requests" {
  name                = "ai-resume-requests"
  resource_group_name = azurerm_resource_group.monitoring.name
  location            = azurerm_resource_group.monitoring.location

  evaluation_frequency = "PT1M"
  window_duration      = "PT5M"

  scopes   = [data.azurerm_application_insights.ai.id]
  severity = 3
  enabled  = true

  display_name = "Resume API requests alert"
  description  = "Alert when /api/getresume is called at least once in 5 minutes"

  criteria {
    query = <<-QUERY
      requests
      | where url contains "/api/getresume"
      | summarize Count = count() by bin(timestamp, 5m)
      | where Count > 0
    QUERY

    time_aggregation_method = "Count"
    threshold               = 0
    operator                = "GreaterThan"

    failing_periods {
      minimum_failing_periods_to_trigger_alert = 1
      number_of_evaluation_periods             = 1
    }
  }

  action {
    action_groups = [azurerm_monitor_action_group.resume_alerts.id]
  }

  auto_mitigation_enabled          = true
  workspace_alerts_storage_enabled = false
}
```

---

# 🧪 Testing the Alert

### 1. Hit the endpoint
```
https://<your-function-app>.azurewebsites.net/api/getresume
```

### 2. Verify logs in Application Insights
Run in **Application Insights → Logs**:

```kql
requests
| order by timestamp desc
| take 20
```

### 3. Check alert history  
Azure Portal → Monitor → Alerts → Fired Alerts

### 4. Receive email notification  
The Action Group sends an email when the alert fires.

---

# 🧹 Troubleshooting & Lessons Learned

### Terraform Drift
- Terraform attempted to destroy a resource group Azure refused to delete  
- Fixed by removing ghost resources from state  
- Imported the existing RG to realign Terraform with Azure  

### Alert Not Firing
Solved by:
- Running KQL queries in **Application Insights Logs**, not Resource Graph  
- Using `contains` instead of `endswith`  
- Ensuring the Function App logs requests correctly  
- Recreating the alert in the portal to validate behaviour  

### State Importing
Used:
```
terraform import azurerm_resource_group.monitoring /subscriptions/<subscription-id>/resourceGroups/rg-resume-monitoring
```

### Key Insight
Monitoring pipelines fail silently unless:
- Logs exist  
- Queries match  
- Alerts evaluate  
- Action groups fire  

This project taught me how to
