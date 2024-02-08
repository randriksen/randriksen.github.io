---
title: "Quickly exanding the booking window of all meeting rooms in Microsoft 365"
author: Ole
tags: PowerShell Microsoft Exchange
categories: PowerShell 
author_profile: true
classes: wide
---

So I got a task to expand the booking window for all the meeting rooms at work.
From the default 180 days to 365 days. 
This isn't anything big or complicated to do. It's an everyday task really. But I was wondering, how short can I make the PowerShell script to do this for all meeting rooms?

Here's what i started out with:
```powershell
$mailboxes = Get-Mailbox -Filter {RecipientTypeDetails -eq 'RoomMailbox'}
foreach ($m in $mailboxes) {
    $m | Set-CalendarProcessing -BookingWindowInDays 365
}
```
It's pretty short, but one line there is unneccessary...

```powershell
foreach ($m in (Get-Mailbox -Filter {RecipientTypeDetails -eq 'RoomMailbox'}) ) {
    $m | Set-CalendarProcessing -BookingWindowInDays 365
}
```
Ok, short, but still, I think i can get it down to just one line...
```powershell
Get-Mailbox -Filter {RecipientTypeDetails -eq 'RoomMailbox'}| % { Set-CalendarProcessing -Identity $_.identity -BookingWindowInDays 365} 
```
That's down to one line... but not really that much shorter.  
Anyway... This solves my problem today :D