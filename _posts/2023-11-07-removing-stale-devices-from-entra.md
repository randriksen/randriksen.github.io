---
title: "Removing stale devices from Entra/Intune"
author: Ole
tags: PowerShell Microsoft Azure Intune MicrosoftGraph EntraID
categories: PowerShell Azure MicrosoftGraph
author_profile: true
classes: wide
---

Today, I continued the cleanup process of our Entra Directory.  
There were about 7500 Stale devices in the directory, and I'm not **that** fond of clicking my mouse.
So I needed a quicker method to deal with the devices.  
And as you've probably figured out already, that means PowerShell.

Todays script is really simple, but it does the job.  
The hardest part was figuring out how to do the date formatting for the filtering.  

Anyway here's a script that deletes any device from entra that hasn't logged in for 180 days.


```powershell
# Get the date 180 days ago
$d = (get-date).AddDays(-180)
# Format the date in "yyyy-MM-ddTHH:mm:ssZ" format
$date =Get-Date $date -Format "yyyy-MM-ddTHH:mm:ssZ"  
# Get all devices that have not signed in since the specified date
$devices = get-mgdevice -filter "ApproximateLastSignInDateTime le $date" -all
# Loop over each device
foreach ($device in $devices) {
    # Remove the device
    Remove-MgDeviceByDeviceId -deviceid $device.deviceid 
}
```