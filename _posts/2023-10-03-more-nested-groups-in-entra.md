---
title: "More nested groups in Entra ID/Azure AD"
Author: Ole
tags: PowerShell Microsoft 
categories: PowerShell 
author_profile: true
classes: wide
---

![Nested Groups](/assets/images/nestedgroups/nestedgroups.png)


About a month ago, I wrote about how I was dealing with the [licensing of nested groups in Entra ID/Azure AD](https://www.randriksen.net/powershell/azure/2023/09/04/dealing-with-azure-licenses-and-nested-groups.html). 
Since then, I've delved deeper into the problem because it's not only a problem for licensing but also for assigning roles in Azure.
Both roles and licenses need to be assigned directly to the group that the user(s) is/are members of. If you have a lot of nested groups, you can't just assign the role/license to the top-level group and expect it to be inherited by the nested groups. This is both a good thing and a bad thing. It's good because it gives you a lot of control over who has what, but it's bad because it's a lot of work to assign the same role/license to a lot of groups.

Some might ask why I chose to deal with the nested groups at all, and don't make a dynamic group based on some attribute in or on-prem AD. And in an ideal world that might be the best solution. But with our current situation with a lot of legacy systems and inherited solutions, it's not an easy option. 
We have so many different departments and locations, that the regular attributes are used for those purposes, and the extensionattributes are already in use for more things than they should be. So for now, the quickest and easiest way to make sure everyone gets the correct licenses and roles is to use the nested groups.

Last time, I made a script that would assign licenses to all the groups where there were user members in a set of nested groups.
Now I've expanded on that script so that it will also assign user roles to enterprise applications. It can also be used to revoke licenses and user roles for enterprise applications. I even made it into a module that you can find on [Powershell Gallery](https://www.powershellgallery.com/packages/MGNestedGroups/)

The module is called MGNestedGroups, and it contains 5 functions:

* Get-MGNestedGroups
* Grant-LicenseToMGSubgroups
* Revoke-LicenseFromMGSubgroups
* Add-MGSubgroupsToEnterpriseApp
* Remove-MGSubgroupsFromEnterpriseApp
  
### Get-MGNestedGroups
```powershell	
function Get-MGSubgroups {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $true)]
        [string]$GroupId
    )

    $group = Get-MgGroup -GroupId $GroupId
    $groups = @()

    if ($group) {
        $subs = Get-MgGroupMember -GroupId $group.Id

        foreach ($sub in $subs) {
            $subDetails = Get-MgUser -UserId $sub.Id -ErrorAction SilentlyContinue

            if ($subDetails) {
                # If $sub is a user, add it to the list of groups
                $groups += $group
            }
            else {
                # If $sub is a group, recursively get subgroups
                $subGroupDetails = Get-MgGroup -GroupId $sub.Id -ErrorAction SilentlyContinue

                if ($subGroupDetails) {
                    $groups += Get-MGSubgroups -GroupId $sub.Id  # Recursively call the function
                }
            }
        }
    }
    
    # Select and return unique group display names and IDs
    $groups = $groups | Select-Object DisplayName, Id | Get-Unique -AsString
    return $groups
}
```
This function will take an Entrada ID group ID as input, and then find all the nested groups with user members, and return them as an array.
The reason it returns all the groups with user members is that there are often groups with subgroups that also have user members.  
An example would be a top group for a department; The deparment head would often be in the top group, while the other users would be split into subgroups based on teams or projects.


### Grant-LicenseToMGSubgroups
```powershell	
function Grant-LicenseToMGSubgroups {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $true)]
        [string]$TopGroupName, # Name of the top-level group
        [Parameter(Mandatory = $true)]
        [string]$LicenseName # SkuPartNumber / license name
    )

    # Get the top-level Azure AD group
    $topGroup = Get-MgGroup -Filter "displayName eq '$TopGroupName'"

    $Sku = Get-MgSubscribedSku -All | Where-Object SkuPartNumber -eq $LicenseName

    if ($topGroup) {
        # Recursively retrieve subgroups (replace with your function name)
        $subgroups = Get-MGSubgroups -GroupId $topGroup.Id

        foreach ($subgroup in $subgroups) {
            Write-Host "Applying license to $($subgroup.DisplayName)"
            # Apply the license to the subgroup using Microsoft Graph API
            Set-MgGroupLicense -GroupId $subgroup.Id -AddLicenses @{SkuId = $Sku.SkuId} -RemoveLicenses @()
        }
    }
    else {
        Write-Host "Top-level group '$TopGroupName' not found."
    }
}
```
This function takes an Entra ID group name and a license SKUpartnumber (like `ENTERPRISEPACK` or `ENTERPRISEPREMIUM`) as input, and then assigns the license to all the nested groups with user members. This is just an improved part of the script from last time.

### Revoke-LicenseFromMGSubgroups
```powershell	
function Revoke-LicenseFromMGSubgroups {
    [CmdletBinding()]
    param (
        [Parameter(Mandatory = $true)]
        [string]$TopGroupName, # Name of the top-level group
        [Parameter(Mandatory = $true)]
        [string]$LicenseName # SkuPartNumber / license name
    )

    # Get the top-level Azure AD group
    $topGroup = Get-MgGroup -Filter "displayName eq '$TopGroupName'"

    if ($topGroup) {
        # Recursively retrieve subgroups (replace with your function name)
        $subgroups = Get-MGSubgroups -GroupId $topGroup.Id

        # Get the specified license
        $sku = Get-MgSubscribedSku -All | Where-Object SkuPartNumber -eq $LicenseName

        foreach ($subgroup in $subgroups) {
            Write-Host "Revoking license from $($subgroup.DisplayName)"
            # Revoke the license from the subgroup using Microsoft Graph API
            Set-MgGroupLicense -GroupId $subgroup.Id -AddLicenses @() -RemoveLicenses $sku.SkuId
        }
    }
    else {
        Write-Host "Top-level group '$TopGroupName' not found."
    }
}
```
This does the opposite of the previous function. It takes an Entra ID group name and a license SKUpartnumber as input, and then removes the license from all the nested groups with user members. 

### Add-MGSubgroupsToEnterpriseApp
```powershell	
function Add-MGSubgroupsToEnterpriseApp {
    [CmdletBinding()]
    param (
        [Parameter()]
        [string] $EnterpriseAppId, # ID of the enterprise application
        [Parameter()]
        [string] $EnterpriseAppName, # Name of the enterprise application
        [Parameter()]
        [string] $TopGroupName, # Name of the top-level group
        [Parameter()]
        [string] $TopGroupID # ID of the top-level group
    )

    process {
        try {
            if ($EnterpriseAppId -eq $null -and $EnterpriseAppName -eq $null) {
                Write-Host "Please specify either the EnterpriseAppId or EnterpriseAppName parameter."
                return
            }
            if ($EnterpriseAppId -eq ""){
                $app = Get-MgApplication -Filter "displayName eq '$EnterpriseAppName'"
            }
            if ($TopGroupID -eq $null -and $TopGroupName -eq $null) {
                Write-Host "Please specify either the TopGroupID or TopGroupName parameter."
                return
            }
            if ($TopGroupID -eq "") {
                $TopGroupID = (Get-MgGroup -Filter "displayName eq '$TopGroupName'").Id
            }

            if ($null -eq $app) {
                $app = Get-MgApplication -ApplicationId $EnterpriseAppId
            }
            if ($null -eq $app) {
                Write-Host "Enterprise application '$EnterpriseAppId' not found."
                return
            }

            $appid = $app.AppId
            $serviceprincipal = Get-MgServicePrincipal -Filter "appId eq '$appid'"
            $approle = $serviceprincipal.approles | Where-Object DisplayName -eq "User"

            $params = @{
                principalId = ""
                resourceId  = $serviceprincipal.id
                appRoleId   = $approle.Id
            }

            Write-Host "TopGroupID: $TopGroupID"

            # Get the subgroup using the Get-Subgroups function (replace with your function name)
            $subgroups = Get-MGSubgroups -GroupId $TopGroupID

            if ($subgroups.Count -eq 0) {
                Write-Host "Subgroups not found or do not have subgroups."
                continue
            }

            # Add each subgroup as an owner or member to the enterprise application
            foreach ($subgroup in $subgroups) {
                $params.principalId = $subgroup.Id
                Write-Host $subgroup
                New-MgGroupAppRoleAssignment -BodyParameter $params -GroupId $subgroup.Id
                Write-Host "Added subgroup '$subgroup.DisplayName' to the enterprise application '$EnterpriseAppId'."
            }
        }
        catch {
            Write-Host "An error occurred: $_"
        }
    }
}
```
This function takes an Entra ID group nameor ID and an Enterprise Application name or ID as input, and then assigns the group to the Enterprise Application, with the "User" role. 

This is the main change to the script from last time. As I've mentioned before the entire AD structure at work is based on a lot of nested groups. These days we are changing out our print solution to a cloud based one, which needs to have the rights assigned to the groups that the users are direct members of to work.  

It required a little bit of backwards thinking to work, since the powershell cmdlet `New-MgGroupAppRoleAssignment` requires three parameters to be set: `principalId`, `resourceId` and `appRoleId`. The `principalId` is the ID of the group that you want to assign to the Enterprise Application. The `resourceId` is the ID of the Service Principal attached to the Enterprise Application. And the `appRoleId` is the ID of the role that you want to assign.
Finding the correct `resourceId` and `appRoleId` wasn't as intuitive as I thought it would be. 
The way I ended up finding them was to first find the `appId` of the Enterprise application, then find the Service Principal with that `appId` by running `Get-MgServicePrincipal -Filter "appId eq '$appid'"`. Then I could find the `appRoleId` by running `$serviceprincipal.approles | Where-Object DisplayName -eq "User"`.
The `principalId` was a bit easier to find, since I already had the group ID from the input.
Interestingly enough the `principalID` is pretty redundant, since you also have to provide the `groupId` as a parameter to the `New-MgGroupAppRoleAssignment` cmdlet. But it's still required.


### Remove-MGSubgroupsFromEnterpriseApp
```powershell
function Remove-MGSubgroupsFromEnterpriseApp {
    [CmdletBinding(SupportsShouldProcess)]
    param (
        [Parameter()]
        [string] $EnterpriseAppId, # ID of the enterprise application
        [Parameter()]
        [string] $EnterpriseAppName, # Name of the enterprise application
        [Parameter()]
        [string] $TopGroupName, # Name of the top-level group
        [Parameter()]
        [string] $TopGroupID # ID of the top-level group
    )

    process {
        try {
            if ($EnterpriseAppId -eq $null -and $EnterpriseAppName -eq $null) {
                Write-Host "Please specify either the EnterpriseAppId or EnterpriseAppName parameter."
                return
            }
            if ($EnterpriseAppId -eq "") {
                $EnterpriseAppId = (Get-MgApplication -ConsistencyLevel eventual -Count appCount -Search "DisplayName:$EnterpriseAppName").Id
            }
            if ($TopGroupID -eq $null -and $TopGroupName -eq $null) {
                Write-Host "Please specify either the TopGroupID or TopGroupName parameter."
                return
            }
            if ($TopGroupID -eq "") {
                $TopGroupID = (Get-MgGroup -Filter "displayName eq '$TopGroupName'").Id
            }
            
            # Get the enterprise application
            $app = Get-mgapplication -ApplicationId $EnterpriseAppId

            if ($null -eq $app) {
                Write-Host "Enterprise application '$EnterpriseAppId' not found."
                return
            }

            $appid = $app.AppId
            $serviceprincipal = Get-MgServicePrincipal -Filter "appId eq '$appid'"
            $approle = $serviceprincipal.approles | where-object DisplayName -eq "User"

            # Get the subgroup using the Get-Subgroups function (replace with your function name)
            $subgroups = Get-MGSubgroups -GroupId $TopGroupID

            if ($subgroups.Count -eq 0) {
                Write-Host "Subgroups not found or do not have subgroups."
                return
            }

            foreach ($s in $subgroups) {
                $approleid = $approle.id
                $approleassignmentid = Get-MgGroupAppRoleAssignment -GroupId $s.Id | Where-Object AppRoleId -eq $approleid
                foreach ($a in $approleassignmentid) {
                    Remove-MgGroupAppRoleAssignment -AppRoleAssignmentId $a.Id -GroupId $s.Id -WhatIf:$WhatIfPreference
                }

                Write-Host "Removed subgroup '$s.DisplayName' from the enterprise application '$EnterpriseAppId'."
            }
        }
        catch {
            Write-Host "An error occurred: $_"
        }
    }
}
```
This function does the opposite of the previous function. It takes an Entra ID group name or ID and an Enterprise Application name or ID as input, and then removes the group from the Enterprise Application roles.

Making this function was sadly not as straight forward as revoking the licenses was, where i could just change the parameters to the same cmdlet.
Here I had to find the `appRoleAssignmentId` for each group, and then remove the `appRoleAssignmentId` for each group. 


### Conclusion
This little project has already saved me a ton of clicking around in the Azure portal, and I hope it can be of use to others as well.
The module is far from perfect, and I'm thinking there are probably other things that can be added to it in the future. If anyone has any suggestions, please let me know, or even better, make a pull request on [GitHub](https://github.com/randriksen/MGNestedGroups)
