---
title: "New version of MGNestedGroups"
Author: Ole
tags: PowerShell Microsoft 
categories: PowerShell 
author_profile: true
classes: wide
---

I've just released a new version of my [MGNestedGroups](https://github.com/randriksen/MGNestedGroups) module.
It's a pretty niche module, but it's something I've found usefull at work, so I've improved it a little bit now.

The only change is that I've imporved the efficiency of the Get-MGSubGroups function.
![slow Get-MGSubGroups](/assets/images/nestedgroups/slowsubgroups.png)

For a fairly common nested group in my organization it used to take about 180 seconds to get the list of subgroups.

![fast Get-MGSubGroups](/assets/images/nestedgroups/fastsubgroups.png)
With the new version it takes about 35 seconds to get the same list of subgroups.

Which I think is a pretty good improvement.

