---
title: "Looking up the command line history in Powershell"
date: 2023-09-10 02:00:00 -0000
author: Ole
tags: PowerShell Microsoft 
categories: PowerShell 
author_profile: true
classes: wide
---


Sometimes when I'm trying to solve a problem with Powershell, I'm not working in an IDE like vscode or ISE, but just in a regular PowerShell window. Which is fine, but when I then later need to find the things I did to solve the problem, I often have to go through the command line history.  That's pretty easy, when you remember something about the commands you were running. You can either use the `up arrow` or `F8` to go through the history, or if you're still in the same session, you can use the `Get-History` cmdlet to get the history. Or you can even search through it by pressing `Ctrl+R` and start typing what you remember of the command.

But what about when you're not in the same session anymore, `Get-History` only works for the current session. And if you need a big block of commands, going back through the history with the arrow keys or `F8` can be a bit tedious.  
So how do you access the command line history from previous sessions?

Well, it's actually not that hard to find it. The Console history is a file in your profile folder, and while it's a little bit deep into the folder structure, it's not that hard to find. The regular path is: `C:\Users\<username>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt`
and if it's not there for some reason, you can find it by running this command in powershell: `Get-PSReadlineOption`
![Get-PSReadlineOption](/assets/images/commandlinehistory/commandlinehistory.png)

So if you want to read the entire history, you can either just open the file in notepad: `notepad (Get-PSReadlineOption).HistorySavePath` or you can use the `Get-Content` cmdlet: `Get-Content (Get-PSReadlineOption).HistorySavePath`

