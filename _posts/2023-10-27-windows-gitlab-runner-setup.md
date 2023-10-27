---
title: "How to set up GitLab Runner on a Windows Server"
Author: Ole
tags:  Microsoft GitLab DevOps PowerShell
categories: PowerShell
author_profile: true
classes: wide
---

When I first started my current job, one of the first things I did was set up a GitLab server. I wanted it to be the central place for version control and CI/CD. I'd used GitLab at my previous job and really liked it, so I wanted to use it again.

What I love about GitLab is how it centralizes automation. You can schedule tasks or trigger them when you push to a repository. Setting up runners for your automation is easy, and you can use GitLab runners on Windows, Linux, or MacOS. You can even run automation in a containerized environment using Docker or Kubernetes.

You could, of course, do Windows automation the old-fashioned way with the task scheduler and PowerShell scripts, but I don't like how it ties you to specific servers. The task scheduler isn't centralized or distributed, and if you only have "script servers," version control becomes a challenge. While there are other software options, they often come with a price tag. GitLab being open source and free is a big plus.

I might go into more detail about how I use GitLab for automation in a future post. But for now, I'll show you how to set up a new GitLab runner server on Windows.

Just a quick note: You should already have a GitLab server installed in your environment. I won't cover that in this post.


## Installing the GitLab Runner on Windows
You need a Windows server. This can be virtually any server, whether new or old, virtual or physical (you can even reuse the old script servers that you're trying to replace). I used freshly installed Windows Server 2022 VM in our on-premises environment.

Next, you should install GitLab Runner on the server. Official installation instructions can be found here: [https://docs.gitlab.com/runner/install/windows.html](https://docs.gitlab.com/runner/install/windows.html). However, you don't necessarily need to follow them because I'll walk you through the installation here.


![Where to find GitLab Runner installation instructions](/assets/images/gitlabrunner/adminrunner.png)
If you navigate to the admin portal of your GitLab server and go to Settings -> CI/CD -> Runners, you'll notice three dots. Clicking on those dots will open a menu, allowing you to select "Show runner installation and registration instructions." Once selected, you'll receive a list of commands for installing and registering a runner. You have the option to install a runner on Linux, MacOS, or Windows. For today's demonstration, we'll choose Windows.

These instructions will include server-specific details, so you should copy the provided commands from there and paste them into a PowerShell window on your Windows server. (But wait until you've finished reading this post!)
![GitLab Runner installation instructions](/assets/images/gitlabrunner/installationinstructions.png)

### Registering the runner
Now, you need to register your runner. You can accomplish this by running the following command in PowerShell: `gitlab-runner.exe register`. This initiates an interactive registration process, during which you'll be prompted to provide the URL of your GitLab server and a registration token. You can locate the registration token in the admin portal of your GitLab server, under Settings -> CI/CD -> Runners.

### Using a service user
If your scripts need to run as a specific user or require impersonation in any of your scripts (as the local system user cannot change user context in any way), you must include something in the installation script, or you'll need to address this later. To run GitLab Runner with a service user, you should add the following to the installation script: `--user $user --password $password`. You can replace `$user` and `$password` with the username and password of the service user you intend to use. The command should look like this:

```powershell
.\gitlab-runner.exe install --user $user --password $password
```

However, this action alone won't be sufficient, as the service won't run successfully. The user also requires the grant of "log-on-as-service" rights. A straightforward way to achieve this is by using a script. I sought assistance from Bing's chatbot and ChatGPT to create one, but both produced code that closely resembled something I had seen before. When it became evident that the generated code was non-functional, I decided to rely on the code I knew would work. This code was sourced from a [Stackoverflow post](https://stackoverflow.com/questions/313831/using-powershell-how-do-i-grant-log-on-as-service-to-an-account), which appears to have drawn inspiration from this [gist](https://gist.github.com/grenade/8519655).

```powershell
   <#
.Synopsis
  Grant logon as a service right to the defined user.
.Description
  Stolen from : https://stackoverflow.com/questions/313831/using-powershell-how-do-i-grant-log-on-as-service-to-an-account
.Parameter computerName
  Defines the name of the computer where the user right should be granted.
  Default is the local computer on which the script is run.
.Parameter username
  Defines the username under which the service should run.
  Use the form: domain\username.
  Default is the user under which the script is run.
.Example
  Usage:
  .\Grant-ServiceUserLogon -computerName hostname.domain.com -username "domain\username"
#>
param(
    [string] $computerName = ("{0}.{1}" -f $env:COMPUTERNAME.ToLower(), $env:USERDNSDOMAIN.ToLower()),
    [string] $username = ("{0}\{1}" -f $env:USERDOMAIN, $env:USERNAME)
  )
  Invoke-Command -ComputerName $computerName -Script {
    param([string] $username)
    $tempPath = [System.IO.Path]::GetTempPath()
    $import = Join-Path -Path $tempPath -ChildPath "import.inf"
    if(Test-Path $import) { Remove-Item -Path $import -Force }
    $export = Join-Path -Path $tempPath -ChildPath "export.inf"
    if(Test-Path $export) { Remove-Item -Path $export -Force }
    $secedt = Join-Path -Path $tempPath -ChildPath "secedt.sdb"
    if(Test-Path $secedt) { Remove-Item -Path $secedt -Force }
    try {
      Write-Host ("Granting SeServiceLogonRight to user account: {0} on host: {1}." -f $username, $computerName)
      $sid = ((New-Object System.Security.Principal.NTAccount($username)).Translate([System.Security.Principal.SecurityIdentifier])).Value
      secedit /export /cfg $export
      $sids = (Select-String $export -Pattern "SeServiceLogonRight").Line
      foreach ($line in @("[Unicode]", "Unicode=yes", "[System Access]", "[Event Audit]", "[Registry Values]", "[Version]", "signature=`"`$CHICAGO$`"", "Revision=1", "[Profile Description]", "Description=GrantLogOnAsAService security template", "[Privilege Rights]", "$sids,*$sid")){
        Add-Content $import $line
      }
      secedit /import /db $secedt /cfg $import
      secedit /configure /db $secedt
      gpupdate /force
      Remove-Item -Path $import -Force
      Remove-Item -Path $export -Force
      Remove-Item -Path $secedt -Force
    } catch {
      Write-Host ("Failed to grant SeServiceLogonRight to user account: {0} on host: {1}." -f $username, $computerName)
      $error[0]
    }
  } -ArgumentList $username
```


If, like me, you accidentally installed the service without the service user, you can simply uninstall the service using the following command: `.\gitlab-runner.exe uninstall`. Afterward, you can reinstall it with the correct command.

Now, you might think that this is all you need to do, and you're ready to use your runner. However, there are a few more steps to make it work.

## Git for Windows: The Missing Piece

First things first, let's talk about Git for Windows. Isn't it a bit curious that when you install a GitLab runner, something that relies entirely on Git binaries, it doesn't come with Git? Maybe it's just me, but...

Anyhow, let's get Git. The latest version is waiting for you [right here](https://git-scm.com/download/win). When you install it, just hit the ground running with the default settings – no need to complicate things.

## Choosing the PowerShell Version

The next decision involves selecting the version of PowerShell you want to use for your automation. I recommend using PowerShell 7 because it's cross-platform and the latest version available. You can find the latest version [here](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows?view=powershell-7.3#msi).

If you opt for PowerShell 7, you won't need to make any changes to the standard GitLab Runner installation.

### Configuring the Runner for PowerShell 5.1

If, like me, you are centralizing many scripts from older servers into GitLab and wish to run them with PowerShell 5.1, you'll need to modify the `config.toml` file for the runner. The `config.toml` file is typically located in the GitLab Runner installation folder, with the default path being: `C:\GitLab-Runner\config.toml`.

The change you need to make is in the line that reads: `shell = "pwsh"`. You should change it to `shell = "powershell"`. Alternatively, you can add `pwsh` as an alias to `powershell.exe`.

This change is necessary on Windows Server 2022, as the default PowerShell installation is 5.1, but it doesn't start with the `pwsh` command; it only starts with `powershell`.

## Installing PowerShell Modules

The next step is to ensure that all the necessary PowerShell modules are installed on the server. You can accomplish this by running the following command in PowerShell: `Install-Module -Name <ModuleName>`. To view a list of all installed PowerShell modules on your server, use this PowerShell command: `Get-Module -ListAvailable`.

It's advisable to install the modules with the `-Scope AllUsers` flag so that they are accessible to all users on the server. If you skip this step, you'll need to install the modules separately for each user running scripts on the server, which is not very efficient.

### Using the PSResourceGet Module

There's a new module in the PowerShell landscape that can expedite module installations, known as the PSResourceGet module. You can find it [here](https://www.powershellgallery.com/packages/Microsoft.PowerShell.PSResourceGet/1.0.0). Install it with the following command: `Install-Module -Name Microsoft.PowerShell.PSResourceGet`.

Once you have the PSResourceGet module, you can install all the required modules using the following command: `Install-PSResource -Name <ModuleName>`. This module is designed to be more efficient than the traditional `Install-Module` command, but I'm still in the early stages of testing it, so I can't provide a definitive assessment of its performance at this time.

### Modules I usually install
- ActiveDirectory
- Az
- Sqlserver
- Microsoft.Graph
- Microsoft.PowerShell.SecretManagement
- Microsoft.PowerShell.SecretStore
  

## Installing RSAT Features

If you intend to use the runner for remote administration of other Windows servers, you should be aware that some PowerShell modules are not included when you use `Install-Module`. Therefore, you'll need to install certain RSAT (Remote Server Administration Tools) features on your server. To do this, use the following PowerShell command: `Install-WindowsFeature -Name <FeatureName>`. To view a list of all the RSAT features installed on your server, you can run the following PowerShell command: `Get-WindowsFeature -name *RSAT*`.

![RSAT](assets/images/gitlabrunner/rsat.png)

### RSAT Features I Typically Install

- RSAT-AD-PowerShell             
- RSAT-DHCP

## Merging It All

Congratulations! By following these steps, you should have a fully operational GitLab runner, all set to power your automation.

But hey, if the manual process is a bit annoying, I also thought so, and made it into a PowerShell script that'll be slightly less annoying to work with:

```powershell	
# Define variables
$url = "your server url"
$regtoken = "your registration token"
$description = "your runner description"
$tags = "your runner tags"
$username = "your service user"
$password = "your service user password"

$serviceuser = $true

$modules = @("Az", "SqlServer", "Microsoft.Graph", "Microsoft.PowerShell.SecretManagement", "Microsoft.PowerShell.SecretStore")
$features = @("RSAT-AD-PowerShell", "RSAT-DHCP")

# Function to grant the "Log on as a service" right
function Grant-ServiceUserLogon {
    <#
.Synopsis
  Grant logon as a service right to the defined user.
.Description
  Stolen from : https://stackoverflow.com/questions/313831/using-powershell-how-do-i-grant-log-on-as-service-to-an-account
.Parameter computerName
  Defines the name of the computer where the user right should be granted.
  Default is the local computer on which the script is run.
.Parameter username
  Defines the username under which the service should run.
  Use the form: domain\username.
  Default is the user under which the script is run.
.Example
  Usage:
  .\Grant-ServiceUserLogon -computerName hostname.domain.com -username "domain\username"
#>
param(
    [string] $computerName = ("{0}.{1}" -f $env:COMPUTERNAME.ToLower(), $env:USERDNSDOMAIN.ToLower()),
    [string] $username = ("{0}\{1}" -f $env:USERDOMAIN, $env:USERNAME)
  )
  Invoke-Command -ComputerName $computerName -Script {
    param([string] $username)
    $tempPath = [System.IO.Path]::GetTempPath()
    $import = Join-Path -Path $tempPath -ChildPath "import.inf"
    if(Test-Path $import) { Remove-Item -Path $import -Force }
    $export = Join-Path -Path $tempPath -ChildPath "export.inf"
    if(Test-Path $export) { Remove-Item -Path $export -Force }
    $secedt = Join-Path -Path $tempPath -ChildPath "secedt.sdb"
    if(Test-Path $secedt) { Remove-Item -Path $secedt -Force }
    try {
      Write-Host ("Granting SeServiceLogonRight to user account: {0} on host: {1}." -f $username, $computerName)
      $sid = ((New-Object System.Security.Principal.NTAccount($username)).Translate([System.Security.Principal.SecurityIdentifier])).Value
      secedit /export /cfg $export
      $sids = (Select-String $export -Pattern "SeServiceLogonRight").Line
      foreach ($line in @("[Unicode]", "Unicode=yes", "[System Access]", "[Event Audit]", "[Registry Values]", "[Version]", "signature=`"`$CHICAGO$`"", "Revision=1", "[Profile Description]", "Description=GrantLogOnAsAService security template", "[Privilege Rights]", "$sids,*$sid")){
        Add-Content $import $line
      }
      secedit /import /db $secedt /cfg $import
      secedit /configure /db $secedt
      gpupdate /force
      Remove-Item -Path $import -Force
      Remove-Item -Path $export -Force
      Remove-Item -Path $secedt -Force
    } catch {
      Write-Host ("Failed to grant SeServiceLogonRight to user account: {0} on host: {1}." -f $username, $computerName)
      $error[0]
    }
  } -ArgumentList $username
}

# Function to install GitLab Runner
function install-GitlabRunner {
    Param (
        [Parameter(Mandatory = $false)]
        [string]$user,
        [Parameter(Mandatory = $false)]
        [string]$password,
        [Parameter(Mandatory = $true)]
        [string]$url,
        [Parameter(Mandatory = $true)]
        [string]$regtoken,
        [Parameter(Mandatory = $false)]
        [string]$tags = "powershell",
        [Parameter(Mandatory = $false)]
        [string]$name = "gitlab-runner",      
        [Parameter(Mandatory = $false)]
        [bool]$serviceuser = $false,
        [Parameter(Mandatory = $false)]
        [string]$installPath = "C:\GitLab-Runner",
        [Parameter(Mandatory = $false)]
        [string]$psversion = "7"

    )

    # Run PowerShell as administrator
    # Create a folder for GitLab Runner
    New-Item -Path $installPath -ItemType Directory

    # Change to the folder
    cd $installPath

    # Download GitLab Runner binary
    Invoke-WebRequest -Uri "https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-windows-amd64.exe" -OutFile "gitlab-runner.exe"

    # Register the runner
    .\gitlab-runner.exe install --user $user --password $password 
    .\gitlab-runner.exe start

    # Register the runner with GitLab
    .\gitlab-runner.exe register --url $url --registration-token $regtoken --executor shell --tag-list $tags --name $name  -n

    if ($psversion -ne "7") {
        # Change the shell to PowerShell 5.1
        $config = Get-Content .\config.toml
        $config = $config -replace "shell = ""pwsh""", "shell = ""powershell"""
        $config | Set-Content .\config.toml
    }
}

# Function to install Git
function install-git {
    # Install Git
    invoke-webrequest -Uri "https://github.com/git-for-windows/git/releases/download/v2.42.0.windows.2/Git-2.42.0.2-64-bit.exe" -OutFile "$env:TEMP\git.exe"
    .$env:TEMP\git.exe /VERYSILENT /NORESTART /NOCANCEL /SP- /CLOSEAPPLICATIONS /RESTARTAPPLICATIONS /COMPONENTS="icons,ext\reg\shellhere,assoc,assoc_sh" /DIR="C:\Program Files\Git"
}

# Function to install PowerShell modules
function install-modules {
    Param (
        [Parameter(Mandatory = $false)]
        [string]$psversion = "7",
        [Parameter(Mandatory = $true)]
        [string[]]$modules
    )
    # Install modules
    
    foreach ($module in $modules) {
        if (!(Get-Module -Name $module -ListAvailable)) {
            Install-Module -Name $module -Scope AllUsers -force 
        }
    }
}

# Function to install Windows features
function install-features {
    Param (
        [Parameter(Mandatory = $true)]
        [string[]]$features
    )
    
    Install-WindowsFeature $features
}

function install-ps7 {
    # Install PowerShell 7
    Invoke-WebRequest -Uri "https://github.com/PowerShell/PowerShell/releases/download/v7.3.8/PowerShell-7.3.8-win-x64.msi" -OutFile "$env:TEMP\pwsh.msi"
    msiexec.exe /i $env:TEMP\pwsh.msi /qn /norestart
}

# Check if a service user is provided
if ($serviceuser) {
    # Grant the "Log on as a service" right to the service user
    Grant-ServiceUserLogon -Username $username
    
    # Install GitLab Runner for the service user
    install-GitlabRunner -user $username -password $password -url $url -regtoken $regtoken  -serviceuser $serviceuser
}
else {
    # Install GitLab Runner without a service user
    install-GitlabRunner -url $url -regtoken $regtoken -serviceuser $serviceuser
}

if ($psversion -eq "7") {
    # Install PowerShell 7
    install-ps7
}
# Install Git, PowerShell modules, and Windows features
install-git
install-modules -modules $modules
install-features -features $features
restart-service gitlab-runner
```


