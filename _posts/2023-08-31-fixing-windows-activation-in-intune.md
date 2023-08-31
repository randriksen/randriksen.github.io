---
title: "Fixing Windows Activation in Intune"
date: 2023-08-31 02:00:00 -0000
author: Ole
tags: powershell microsoft intune 
categories: powershell intune
---

I'm currently working on a project where we're migrating everything we can of client computers to Intune.
The process has been to take in all the client computers, grab all the autopilot-info and reinstall them from SCCM with a blank Windows 11 ready for Out-of-box-experience.
In this process we accidentally installed them with a KMS license key. Which is a problem, since we're not going to use KMS for these clients.
So we needed to change the license key to the correct one, and activate Windows.

I found a few different ways to do this, but the ones that seemed easy and elegant (like this one: ![https://www.linkedin.com/pulse/windows-10-edition-upgrade-via-intune-noel-fairclough/](https://www.linkedin.com/pulse/windows-10-edition-upgrade-via-intune-noel-fairclough/)), just didn't work.

So after a little bit of trial and error, I found something that works for us:
It's powershell scripts that we run in Intune, first a detection script:
'''powershell
# First get the active license object from the computer
$license = get-ciminstance softwarelicensingproduct | where-object {$_.PartialProductKey}
# check if the license object is a KMS license
if ($license.description -like "*KMS*") {
    # if it is a KMS license, exit with error code 1
 exit 1
} else {
    # if it is not a KMS license, exit with error code 0
 exit 0
}
'''

And then a remediation script:
'''powershell
# Remove the active product key with slmgr (suppressed from output)
slmgr //b /upk
# Remove the name of the KMS server 
slmgr //b /ckms
# Add the new product key
slmgr //b /ipk <product key>
'''


If you need to do this:
* Go to Intune -> Devices -> Windows -> Scripts
* Create new script
* Give it a name and description
![Intune script](/pictures/windowsactivation/intunewindowsactivation.png)
* Add the detection script
* Add the remediation script
* Leave everything else as default
* Choose the scope
* Assign it to the computers you want to run it on
* Save it
  
That's it. 


