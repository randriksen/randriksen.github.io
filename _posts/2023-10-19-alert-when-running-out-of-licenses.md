---
title: "Alerting when running out of licenses"
Author: Ole
tags: PowerShell Microsoft Graph Microsoft365
categories: PowerShell 
author_profile: true
classes: wide
---

![running out of licenses](/assets/images/runningoutoflicenses/runningoutoflicenses.png)
Lately, we've been experiencing a shortage of licenses for some of our Microsoft 365 services.  
It's not a significant problem, but the absence of any alerting mechanism made it a bit of a hassle when new users couldn't obtain the licenses they needed.

To address this issue, I've decided to implement two different methods for alerting us.

The first method involves using a Log search alert in Azure Monitor. It relies on a simple Kusto Query Language (KQL) query to check for any license assignment failures in the last 30 minutes:
    
```kql
AuditLogs | 
where Result == "failure" | 
where ResultReason == "Microsoft.Online.Provisioning.SubscriptionManagement.SubscriptionFullException
```

While this method works, it lacks specificity. It will trigger an alert for any license assignment failure, not just when we run out of licenses.  
It's certainly a report we want, but it doesn't provide us with proactive insights.

For the second alert, I've developed a straightforward PowerShell script that depends on the Microsoft Graph PowerShell SDK:
```powershell
Connect-MGGraph

# Get a list of subscribed SKUs (licenses)
$licenses = Get-MgSubscribedSku -All

# Define an array of license SKUs to exclude from further analysis
$exclude = @("VISIOCLIENT", "PROJECTPREMIUM", "MEETING_ROOM", "EMSPREMIUM_EDU_FACULTY", "PROJECTPROFESSIONAL", "RMSBASIC")

# Filter licenses that are running out
$runningOut = $licenses | where-object SkuPartNumber -NotIn $exclude | select SkuPartNumber, ConsumedUnits, @{label="Enabled"; expression={$_.PrepaidUnits.Enabled}} | where-object {$_.ConsumedUnits -gt ($_.enabled-3)}

# Check if there are licenses running out
if (!$null -eq $runningOut) {
    # If there are licenses running out do something
    write-host $runningOut
}

```

This script will connect to the Microsoft Graph and retrieve a list of all the licenses we have. It will then filter out licenses for which we don't need alerts, such as Visio, Project, Meeting room licenses, etc., as we purchase these licenses on a 1-to-1 basis.  
Finally, it will filter the licenses with fewer than 3 available for assignment.
If any licenses are running out, the script will take action. In the example, it simply writes to the host, but you can configure it to send an email, a message in Teams, or any other action you prefer.

I have scheduled this script to run every 30 minutes, ensuring that we receive alerts whenever we need to acquire more licenses.