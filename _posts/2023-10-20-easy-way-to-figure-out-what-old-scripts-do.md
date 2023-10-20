---
title: "An easy way to figure out what old scripts actually do, document them and maybe even improve them"
Author: Ole
tags: PowerShell Microsoft ChatGPT
categories: PowerShell 
author_profile: true
classes: wide
---

In most of my jobs in the last 12 years, I've encountered situations where my predecessors had already left before I started, and I had to search far and wide for documentation on how they had set up the systems.  
Sometimes the documentation was available, and sometimes it wasn't. At times, it was fairly up-to-date, but most of the time, it was quite outdated.

People document their processes differently. I've met individuals who have crafted fantastic documentation, and I've encountered people whose documentation consisted solely of screenshots depicting their actions and nothing more.

So, it's safe to say that I've had to decipher the purpose and functionality of old scripts in scheduled tasks on numerous occasions.  
Often, this has entailed a lengthy process of reading through the scripts and attempting to understand the purpose of uncommented code.  
Sometimes, the only way to gain clarity has been to test the scripts in a lab environment, on a test server, or even in production (typically with the "-whatif" flag activated or with any data-altering functionality disabled) just to observe the actual outcome.

However, these days, we have new tools that simplify this process significantly. With tools like ChatGPT, all you need to do is paste the scripts and instruct it to comment on their functionality and how to use them.

You can even utilize ChatGPT to enhance the script, document it and make it more readable and aligned with best practices.

For instance, consider this small script:

```powershell
#this script was for https://techcommunity.microsoft.com/t5/windows-powershell/powershell-script/m-p/3947118/highlight/false#M7212
#someone wanted a script to send an email when a user was locked out of their account

#make a scheduled task triggered on the event and then have it run this script


$alert = Get-EventLog -LogName security -instanceid 4740 -Newest 1
$body = $alert.message



#Send email with the report
$smtpServer = "yourmailserver"
$smtpPort = 25
#$smtpUsername = "email address removed for privacy reasons"
#$smtpPassword = "your_email_password"
             
$to = "sendto"
$from = "sendrom"
$event = $alert.entrytype
$time = $alert.TimeGenerated
$subject = "$event - $time"
             
$message = New-Object System.Net.Mail.MailMessage $from, $to
$message.Subject = $subject
$message.Body = $body
#$message.IsBodyHtml = $true
             
$smtp = New-Object System.Net.Mail.SmtpClient $smtpServer, $smtpPort
#$smtp.EnableSsl = $true
#$smtp.Credentials = New-Object System.Net.NetworkCredential $smtpUsername, $smtpPassword

$smtp.Send($message)
```

If I send it through ChatGPT with this prompt: "Please comment and document this script: {script}."

I receive the following result:

![ChatGPT](/assets/images/easy-way/ChatGPT.png)


```powershell
# Script for sending an email notification when a user is locked out of their account.
# This script is intended to be used as part of a scheduled task triggered by Event ID 4740 in the Security Event Log.

# Retrieve the most recent event log entry for Event ID 4740 (account lockout).
$alert = Get-EventLog -LogName Security -InstanceId 4740 -Newest 1
$body = $alert.Message

# Configure email settings.
$smtpServer = "yourmailserver"  # Replace with your SMTP server's address
$smtpPort = 25  # Replace with your SMTP server's port (commonly 25 for non-secure connections)
$to = "sendto"  # The recipient's email address
$from = "sendrom"  # The sender's email address (your organization's email address)

# Build the email subject using event type and timestamp
$event = $alert.EntryType  # Security event type (e.g., Information, Warning, Error)
$time = $alert.TimeGenerated  # Timestamp of the event
$subject = "$event - $time"  # Combine event type and timestamp for the subject

# Create a new email message
$message = New-Object System.Net.Mail.MailMessage $from, $to
$message.Subject = $subject
$message.Body = $body  # The event message content
# Optionally, set $message.IsBodyHtml = $true if the content is HTML-formatted

# Create an SMTP client for sending the email
$smtp = New-Object System.Net.Mail.SmtpClient $smtpServer, $smtpPort
# If your SMTP server requires authentication, uncomment and configure the following lines:
# $smtp.EnableSsl = $true  # Enable SSL if required
# $smtp.Credentials = New-Object System.Net.NetworkCredential $smtpUsername, $smtpPassword

# Send the email
$smtp.Send($message)

```

You can even take it a step further and ask ChatGPT to enhance the script for you, making it more readable and aligned with best practices. Simply use this prompt:

"Please document and improve the script and make it conform to best practices."

This prompt will guide ChatGPT to optimize the script for you.

```powershell
<#
.SYNOPSIS
This script sends an email notification when a user is locked out of their account based on Event ID 4740 from the Windows Security Event Log.

.DESCRIPTION
The script retrieves the most recent security event with Event ID 4740, extracts the relevant information, and sends an email notification with the event details.

.NOTES
- Customize the script with your email server settings, including SMTP server address, port, sender and recipient email addresses.
- Uncomment and provide necessary SMTP authentication settings and SSL configuration if required.
- Ensure you run the script with appropriate permissions and scheduled task configuration.
#>

function Send-Email {
    param (
        [string]$to,
        [string]$from,
        [string]$subject,
        [string]$body
    )

    try {
        $message = New-Object System.Net.Mail.MailMessage $from, $to
        $message.Subject = $subject
        $message.Body = $body
        # $message.IsBodyHtml = $true  # Uncomment if the email body contains HTML content

        $smtpServer = "yourmailserver"  # Replace with your SMTP server address
        $smtpPort = 25  # Replace with the appropriate SMTP port
        $smtp = New-Object System.Net.Mail.SmtpClient $smtpServer, $smtpPort
        # $smtp.EnableSsl = $true  # Uncomment if your SMTP server requires SSL
        # $smtp.Credentials = New-Object System.Net.NetworkCredential $smtpUsername, $smtpPassword  # Uncomment if authentication is required

        $smtp.Send($message)
        Write-Host "Email sent successfully."
    } catch {
        Write-Host "An error occurred while sending the email: $_"
    }
}

function Get-LatestLockoutEvent {
    try {
        $event = Get-EventLog -LogName Security -InstanceId 4740 -Newest 1
        return $event
    } catch {
        Write-Host "An error occurred while retrieving the latest lockout event: $_"
        return $null
    }
}

# Main script logic
$latestLockoutEvent = Get-LatestLockoutEvent

if ($latestLockoutEvent -ne $null) {
    $event = $latestLockoutEvent.EntryType
    $time = $latestLockoutEvent.TimeGenerated
    $subject = "$event - $time"
    $body = $latestLockoutEvent.Message

    $to = "sendto"  # Replace with the recipient's email address
    $from = "sendfrom"  # Replace with the sender's email address

    Send-Email -to $to -from $from -subject $subject -body $body
}

```

This little trick has saved me a lot of time, and I'm sure it will save me even more in the future.

But there are some things to keep in mind when using ChatGPT like this.

First, remember that the scripts you send to ChatGPT are stored on their servers and used to train the AI. So, if you're dealing with sensitive data, it's best not to use ChatGPT for this purpose. Make sure the scripts you send don't have any sensitive information. If you think a script reveals too much about your environment, consider modifying it before asking ChatGPT or another AI/LLM to comment and document it.

Second, ChatGPT can sometimes provide results that are incorrect or don't make sense. For example, it might change a working script in a way that introduces errors. In a previous example, it replaced Get-EventLog with Get-WinEvent for supposed efficiency. Both cmdlets work, but they work differently. And when ChatGPT changed it, it also wanted to run `Get-WinEvent` with the parameter `-FilterHashTable @{LogName='Security';ID=4740}`, and on Windows 11 at least, `-FilterHashTable` is not a valid parameter for `Get-WinEvent`.

So, be sure to test your scripts after using ChatGPT, as they might not work correctly.

This was all done with the free version of ChatGPT, so it was only  ChatGPT 3.5. It would probably be even better with ChatGPT 4.0.
