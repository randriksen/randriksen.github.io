---
title: "Looking up the command line history in Powershell"
date: 2023-09-10 02:00:00 -0000
author: Ole
tags: PowerShell Microsoft 
categories: PowerShell 
author_profile: true
classes: wide
---

Sometimes when I'm trying to solve a problem with Powershell, I'm not working in an IDE like vscode or ISE, but just in a regular PowerShell window. Which is fine, but when I then later need to find the things I did to solve the problem, I often have to go through the command line history.  That's pretty easy, when you remember something about the commands you were running. You can either use the `up arrow` or `F8` to go through the history, or if you're still in the same session, you can use the `Get-History` cmdlet (just `h` if you want to type less) to get the history. Or you can even search backwards through it by pressing `Ctrl+R` and start typing what you remember of the command, or `Ctrl-S` to search forward. And if you want to run a command from the history, you can use the `Invoke-History` cmdlet (just `r` if you want to type less).

But what about when you're not in the same session anymore, `Get-History` only works for the current session. And if you need a big block of commands, going back through the history with the arrow keys or `F8` can be a bit tedious.  
So how do you access the command line history from previous sessions?

Well, it's actually not that hard to find it. The Console history is a file in your profile folder, and while it's a little bit deep into the folder structure, it's not that hard to find. The regular path is: `C:\Users\<username>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt`
and if it's not there for some reason, you can find it by running this command in powershell: `Get-PSReadlineOption`
![Get-PSReadlineOption](/assets/images/commandlinehistory/commandlinehistory.png)

So if you want to read the entire history, you can either just open the file in notepad: `notepad (Get-PSReadlineOption).HistorySavePath` or you can use the `Get-Content` cmdlet: `Get-Content (Get-PSReadlineOption).HistorySavePath`

But, in a fit of creativity, I decided that I want more: I want commands that works like Get-History and Invoke-History, but for the entire commandline history. Thus I've made these monstrosities: Get-ExtendedHistory and Invoke-ExtendedHistory.
They might not be pretty, and they have a helper function called Split-History, but for my purposes they mostly work. Though it got a bit confusing to run when I basically tested them on themself. 

Anyway, here they are:

```powershell
# Define a function called Split-History
function Split-History {
    # Read the content of the PowerShell history file
    $historyContent = Get-Content (Get-PSReadlineOption).HistorySavePath

    # Initialize arrays to store commands and the current command being processed
    $commands = @()
    $currentCommand = ""

    # Loop through each line in the history file
    foreach ($line in $historyContent) {
        # Check if the line ends with "``" (backtick), indicating a continuation of a multiline command
        if ($line.EndsWith("``")) {
            # Replace the backtick with line change and append the line to the current command
            $currentCommand += $line.Replace("``", "`n")
        } else {
            # If the line doesn't end with a backtick, it's a standalone command
            $currentCommand += $line
            # Add the complete command to the list of commands, trimming any leading/trailing whitespace
            $commands += $currentCommand.Trim()
            # Reset the current command string
            $currentCommand = ""
        }
    }

    # Add the last command if it exists
    if ($currentCommand -ne "") {
        $commands += $currentCommand.Trim()
    }

    # Return the array of individual commands
    return $commands
}

# Define a function called get-extendedhistory
function Get-ExtendedHistory {
    # Call the Split-History function to split the history into coherent commands
    $hist = split-history

    # Output headers for displaying command history
    write-host "Id`tCommand"
    write-host "-----------------------"

    # Loop through the commands in the history
    for ($i = 0; $i -lt $hist.Length; $i++) {
        # If a command is very long (over 200 characters), add separators for readability
        if ($hist[$i].length -gt 200) {
            write-host "`n--------------------------------------------------------------------------------------------"
        }

        # Output the command's index and the command itself
        write-host ($i) "`t" $hist[$i]

        # If a command is very long (over 200 characters), add separators for readability
        if ($hist[$i].length -gt 200) {
            write-host "--------------------------------------------------------------------------------------------`n"
        }
    }
}

# Define a function called invoke-extendedhistory
function Invoke-ExtendedHistory {
    param([int]$i)
    # Call the Split-History function to split the history into coherent commands
    $hist = split-history
    # Retrieve the command at the specified index
    $line = $hist[$i]
    # Execute the command using Invoke-Expression
    Invoke-Expression -Command $line
}

```