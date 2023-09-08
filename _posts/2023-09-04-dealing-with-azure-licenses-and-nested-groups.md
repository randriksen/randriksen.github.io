---
title: "Dealing with Azure licenses and nested groups"
date: 2023-09-04 02:00:00 -0000
author: Ole
tags: PowerShell Microsoft Azure Intune
categories: PowerShell Azure
author_profile: true
classes: wide
---

*updated 06.09.2023*

One of my current projects at work is to migrate as many as possible of our clients from on-prem clients, to Intune managed clients.
We're also starting to utilize more of the cloud services that Microsoft offers, and for the Intune managed clients we're starting to utilize Microsoft defender for endpoint. And with this comes the need for licenses.  
And as anyone who has worked with licenses knows, that's always the fun part.  
Now, it's not exactly hard to deal with the licenses, but it can be very tedious and time consuming if you do it explicitly for each user.  
Luckily you don't need to do that. You can use groups to assign licenses to users. The bad thing with this is that you can't use nested groups. In AzureAD you need to assign the licenses to the groups that the users are in, and not the groups that the groups are in. (very annoying)  
And if you're coming from an on-prem AD scructure, you might be using nested groups A LOT.

In my case, we're the IT department for a whole bunch of schools. Each school has several classes, and each class has several students. 
So the case is this: All students need to get their **Microsoft 365 A5 Security for student use benefits** license. All students are in a group for their class, all classes in a "all students" group for their school, and all these groups are in a "all students" group for the entire municipality.
If I could have just assigned the licenses to this group, that would have been awesome. But sadly, that doesn't work.  
Instead, I have to assign the licenses to each class group.

Now there are at least 4 ways to do this:

## First the "wrong way":

* Go into each group in AzureAD
![assigning licenses the "wrong way"](/assets/images/azure-licenses/wrong-way.png)
* Click on "Licenses"
* Click on "Assignments"
* Click on "Add assignments"
* Choose the licenses you want to assign
* Click "Assign"

The reason this is the "wrong way" is that it takes too long, and you might lose track of which groups you've assigned the licenses to.
Lots of clickling, hard to keep track


## The "right way" (but still not the best way):
* Go into licenses in AzureAD
![assigning licenses the "right way"](/assets/images/azure-licenses/right-way.png)
* Click on "All products"
* Click on the license you want to assign
* Click on "Assign"
* Click on "Users and groups"
* Click on "Add assignments"
* Choose the groups you want to assign the license to
* Click "Assign"


This is better, but still not great. It's still a lot of clicking, and it's easier to keep track over which groups you've assigned the licenses to, but it's still not great. And if you click wrong during the selection of groups or users, it will blank out your selection, and you have to start over.


### The dynamic groups way (but with lots of limitation):
This is basically the same as the "right way" but instead of using regular groups, you use a dynamic group. 
[![this is a picture form microsoft learn](https://learn.microsoft.com/en-us/azure/active-directory/enterprise-users/media/groups-dynamic-rule-member-of/member-of-diagram.png)](https://learn.microsoft.com/en-us/azure/active-directory/enterprise-users/media/groups-dynamic-rule-member-of/member-of-diagram.png)
* Go into AzureAD
* Click on "Groups"
* Click on "New group"
* Choose "Security"
* Give it a name
* Click on "Add dynamic query"
* MemberOf isn't yet supported in the rule builder.  
  Select Edit to write the rule in the Rule syntax box.  
  Example user rule: user.memberof -any (group.objectId -in ['groupId', 'groupId'])  
* Choose the groups you want to assign the license to.
* Click "Select"
* Click "Create"

This will give you a new group that has all the users that you need to assign the licenses to, as direct members, when the other groups change members, this one will too. So you only need to assign the licenses to this group.
This would probably have been the smartest way to do it, but it has a lot of limitations due to it being a feature in [preview](https://learn.microsoft.com/en-us/azure/active-directory/enterprise-users/groups-dynamic-rule-member-of)
here's the official list of limitations:
## Preview limitations
* Each Azure AD tenant is limited to 500 dynamic groups using the memberOf attribute. memberOf groups do count towards the total dynamic group member quota of 5,000.
* Each dynamic group can have up to 50 member groups.
* When adding members of security groups to memberOf dynamic groups, only direct members of the security group become members of the dynamic group.
* You can't use one memberOf dynamic group to define the membership of another memberOf dynamic groups. For example, Dynamic Group A, with members of group B and C in it, can't be a member of Dynamic Group D).
* MemberOf can't be used with other rules. For example, a rule that states dynamic group A should contain members of group B and also should contain only users located in Redmond will fail.
* Dynamic group rule builder and validate feature can't be used for memberOf at this time.
* MemberOf can't be used with other operators. For example, you can't create a rule that states “Members Of group A can't be in Dynamic group B.”
Also you need to know the group ID's of the groups you want to pull the members from, and that makes changing this group in the future a bit annoying, since you have to check which groups you already have in the query, and then add the new ones. And if you have a lot of groups, it's easy to lose track of which ones you've already added.

I played around with a way to make dynamic groups out of nested groups with powershell. It's in the [github repo](https://github.com/randriksen/powershell). It doesn't work in my case due to the limitations on amounts of groups that can be used in a dynamic group, but it might work for you, if you have fewer groups to pull users from.

### The powershell way:
This is the type of task I imagine that I'll have to do again and again (as long as I haven't found a better way to do it dynamically), so I'll automate it, or at least make a tool that I can use myself or get one of my colleagues to use and not have to worry about things being done wrong, or forgotten.
So I sat down and spent probably more time than it would have taken to do it manually, and made a powershell script that does it for me.
It's far from perfect, and I'll probably come with some improvements later, but it solves the problem i have right now.
It's implemented with the Microsoft Graph API, so you'll need to install the [Microsoft.Graph](https://www.powershellgallery.com/packages/Microsoft.Graph/1.6.2) module from the powershell gallery.

The script takes two parameters: A group name and a license name/SkuPartNumber.
The benefit of this way of doing it is that it will find all subgroups with direct user members and license them as well, so if one group has some direct members and also subgroups, it will make sure everyone inside are licensed.u

Also adding it to my [github repo](https://github.com/randriksen/powershell) for random powershell stuff

```powershell
# Connect to Microsoft Graph with specified scopes
Connect-MgGraph -Scopes Directory.Read.All,Directory.ReadWrite.All,Organization.Read.All,Organization.ReadWrite.All

# Define a function to create license parameters based on a license name/SkuPartNumber 
# This can be found by running Get-MgSubscribedSku
function make-licenceparameters ($license) {
    # Get the subscribed SKU matching the given license
    $sku = Get-MgSubscribedSku | where skupartnumber -eq $license
    
    # Define license parameters
    $params = @{
        addLicenses = @(
            @{
                disabledPlans = @(
                    # List of disabled plans (if any)
                )
                skuId = $sku.skuid
            }
        )
        removeLicenses = @(
            # List of licenses to remove (if any)
        )
    }
    return $params
}

# Define a function to recursively get subgroups with user members of a specified group
function Get-Subgroups {
    param (
        [string]$GroupId
    )

    $group = Get-MgGroup -GroupId $GroupId
    $groups = @()

    if ($group) {
        $subs = Get-MgGroupMember -groupid $group.Id

        foreach ($sub in $subs) {
            $subDetails = Get-MgUser -userid $sub.Id -ErrorAction SilentlyContinue


            if ($subDetails) { #if $sub is a user, add it to the list of groups
                $groups += $group
                break
            } else { # if $sub is a group, recusevly get subgroups
                $subGroupDetails = Get-MgGroup -groupId $sub.Id -ErrorAction SilentlyContinue

                if ($subGroupDetails) {
                    $groups += Get-Subgroups -GroupId $sub.Id
                }
            }
        }
    }

    return $groups 
}

# Define a function to apply a license to leaf groups in a hierarchy
function license-leafgroups {
    param (
        [string]$topGroupName, #name of the top-level group
        [string]$licenseName #SkuPartNumber / license name
    )
    
    # Get the top-level Azure AD group
 	$topGroup = Get-MgGroup -Filter "displayName eq '$topGroupName'"
	
    # Recursively retrieve subgroups
    $subgroups = Get-Subgroups ($topGroup.ObjectId)

    # Generate license parameters for the specified license
    $licenseParameters = make-licenceparameters $licenseName

    foreach ($sub in $subgroups) {
        Write-Host $sub.DisplayName
        # Apply the license to the subgroup using Microsoft Graph API
        Set-MgGroupLicense -GroupId $sub.Id -BodyParameter $licenseParameters -ErrorAction SilentlyContinue
    }
}
```

And then you can use it like this:
```powershell
license-leafgroups -topGroupName "All students" -licenseName "IDENTITY_THREAT_PROTECTION_STUUSEBNFT"
```

That's it, hopefully someone can steal some code from me and save them a couple of minutes :D
