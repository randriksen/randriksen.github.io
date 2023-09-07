---
title: "Tagging untagged autopilot devices in Intune"
date: 2023-09-07 02:00:00 -0000
author: Ole
tags: Powershell Microsoft Intune
categories: Powershell Intune
author_profile: true
---

The other day I accidentally imported 800+ devices into Intune autopilot, without having tags in the CSV files. 
The way I should have imported the devices, was to have a column in the CSV file with a Group Tag column, like this:
```csv
Device Serial Number,Windows Product ID,Hardware Hash,Group Tag
#serial,,"#long hash",#Tag 
```
|Device Serial Number|Windows Product ID           |Hardware Hash|Group Tag                                    |
|--------------------|-----------------------------|-------------|---------------------------------------------|
|#serial             |                             |#long hash   |#Tag                                         |

But my csv file didn't have the Group Tag column, so all the devices were imported without tags.
If I had given this a little bit more thought before I imported the devices, and imported them with the PowerShell cmdlet: `Import-AutopilotCSV` I could have used the `-GroupTag` parameter to add the tag, but I didn't. I used the Intune portal to import the devices, and there is no way to add the tag there.

Now, dealing with this in the Intune portal is ok if you need to tag just a couple of devices, but once its more than a couple it isn't ok anymore.

### Powershell to the rescue!

Luckily, PowerShell is one of the best friends you can have when it comes to dealing with large sets of data,
and the Microsoft Graph API is a great way to interact with Intune.

This little script will connect to the Microsoft Graph API, and get a list of all the Autopilot devices in Intune. It will then filter out all the devices that do not have a Group Tag or a Purchase Order Identifier, and then loop through the devices and set the Group Tag to a value you define.

```powershell	
# Check if the mggraph module is already installed
if (-not (Get-Module -ListAvailable -Name mggraph)) {
    # If not installed, install the mggraph module
    Install-Module -Name mggraph -Force
}

# Import the mggraph module
Import-Module -Name mggraph

# Connect to the Microsoft Graph API with the required scopes
Connect-MgGraph -Scopes DeviceManagementServiceConfig.Read.All, DeviceManagementServiceConfig.ReadWrite.All

# Get a list of Autopilot devices
$autopilotDevices = Get-AutopilotDevice

# Filter devices that do not have a groupTag or purchaseOrderIdentifier
$devicesWithoutTagsOrPOI = $autopilotDevices | Where-Object { $_.GroupTag -eq $null -or $_.PurchaseOrderIdentifier -eq $null }

# Define the groupTag value to assign to devices without tags
$newGroupTag = "Tag"

# Loop through devices without tags or purchase order identifiers and set the groupTag
foreach ($device in $devicesWithoutTagsOrPOI) {
    Set-AutopilotDevice -DeviceId $device.DeviceId -GroupTag $newGroupTag
}

# Output a message indicating the completion of the operation
Write-Host "GroupTag has been set for devices without tags or purchase order identifiers."
```




The script is also available in my [github repo](https://github.com/randriksen/powershell)