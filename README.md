# learn-wsus
how to wsus
### how to tar in windows
```bash
tar -czf C:\temp\WsusContent_%date:~4,2%-%date:~7,2%-%date:~10,4%.tar.gz -C C:\WSUS_database WsusContent
```
### how to export wsus dynamic 
```bash
C:\"Program Files\Update Services"\Tools\WsusUtil.exe export "C:\temp\export_%date:~4,2%-%date:~7,2%-%date:~10,4%.xml.gz" "C:\temp\export_%date:~4,2%-%date:~7,2%-%date:~10,4%.log"
```
```bash
Explanation:
%date:~4,2%: Extracts the month from the date string (starting from position 4, taking 2 characters).
%date:~7,2%: Extracts the day from the date string (starting from position 7, taking 2 characters).
%date:~10,4%: Extracts the year from the date string (starting from position 10, taking 4 characters).
This approach will use the date in MM-DD-YYYY format for both the export and date.log filenames, for example: export02-13-2025.xml.gz and date02-13-2025.log.
```
### How to use Powershell script
```bash
.EXAMPLE
# Use with remote server IP, port and SSL
.\ImportUpdateToWSUS.ps1 -WsusServer 127.0.0.1 -PortNumber 8531 -UseSsl -UpdateId 12345678-90ab-cdef-1234-567890abcdef

.EXAMPLE
# Use with remote server Name, port and SSL
.\ImportUpdateToWSUS.ps1 -WsusServer WSUSServer1.us.contoso.com -PortNumber 8531 -UseSsl -UpdateId 12345678-90ab-cdef-1234-567890abcdef

.EXAMPLE
# Use with remote server IP, defaultport and no SSL
.\ImportUpdateToWSUS.ps1 -WsusServer 127.0.0.1  -UpdateId 12345678-90ab-cdef-1234-567890abcdef

.EXAMPLE
# Use with localhost default port
.\ImportUpdateToWSUS.ps1 -UpdateId 12345678-90ab-cdef-1234-567890abcdef

.EXAMPLE
# Use with localhost default port, file with updateID's
.\ImportUpdateToWSUS.ps1 -UpdateIdFilePath .\file.txt
```
Save as ImportUpdateToWSUS.ps1
```bash
<#
.SYNOPSIS
Powershell script to import an update, or multiple updates into WSUS based on the UpdateID from the catalog.

.DESCRIPTION
This script takes user input and attempts to connect to the WSUS server.
Then it tries to import the update using the provided UpdateID from the catalog.

.INPUTS
The script takes WSUS server Name/IP, WSUS server port, SSL configuration option and UpdateID as input. UpdateID can be viewed and copied from the update details page for any update in the catalog, https://catalog.update.microsoft.com.

.OUTPUTS
Writes logging information to standard output.

.EXAMPLE
# Use with remote server IP, port and SSL
.\ImportUpdateToWSUS.ps1 -WsusServer 127.0.0.1 -PortNumber 8531 -UseSsl -UpdateId 12345678-90ab-cdef-1234-567890abcdef

.EXAMPLE
# Use with remote server Name, port and SSL
.\ImportUpdateToWSUS.ps1 -WsusServer WSUSServer1.us.contoso.com -PortNumber 8531 -UseSsl -UpdateId 12345678-90ab-cdef-1234-567890abcdef

.EXAMPLE
# Use with remote server IP, defaultport and no SSL
.\ImportUpdateToWSUS.ps1 -WsusServer 127.0.0.1  -UpdateId 12345678-90ab-cdef-1234-567890abcdef

.EXAMPLE
# Use with localhost default port
.\ImportUpdateToWSUS.ps1 -UpdateId 12345678-90ab-cdef-1234-567890abcdef

.EXAMPLE
# Use with localhost default port, file with updateID's
.\ImportUpdateToWSUS.ps1 -UpdateIdFilePath .\file.txt

.NOTES  
# On error, try enabling TLS: https://learn.microsoft.com/mem/configmgr/core/plan-design/security/enable-tls-1-2-client

# Sample registry add for the WSUS server from command line. Restarts the WSUSService and IIS after adding:
reg add HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\.NETFramework\v4.0.30319 /V SchUseStrongCrypto /T REG_DWORD /D 1

## Sample registry add for the WSUS server from PowerShell. Restarts WSUSService and IIS after adding:
$registryPath = "HKLM:\Software\Microsoft\.NETFramework\v4.0.30319"
$Name = "SchUseStrongCrypto"
$value = "1" 
if (!(Test-Path $registryPath)) {
    New-Item -Path $registryPath -Force | Out-Null
}
New-ItemProperty -Path $registryPath -Name $name -Value $value -PropertyType DWORD -Force | Out-Null
Restart-Service WsusService, w3svc

# Update import logs/errors are under %ProgramFiles%\Update Services\LogFiles\SoftwareDistribution.log

#>

param(
    [Parameter(Mandatory = $false, HelpMessage = "Specifies the name of a WSUS server, if not specified connects to localhost")]
    # Specifies the name of a WSUS server, if not specified connects to localhost.
    [string]$WsusServer,

[Parameter(Mandatory = $false, HelpMessage = "Specifies the port number to use to communicate with the upstream WSUS server, default is 8530")]
    # Specifies the port number to use to communicate with the upstream WSUS server, default is 8530.
    [ValidateSet("80", "443", "8530", "8531")]
    [int32]$PortNumber = 8530,

[Parameter(Mandatory = $false, HelpMessage = "Specifies that the WSUS server should use Secure Sockets Layer (SSL) via HTTPS to communicate with an upstream server")]
    # Specifies that the WSUS server should use Secure Sockets Layer (SSL) via HTTPS to communicate with an upstream server.  
    [Switch]$UseSsl,

[Parameter(Mandatory = $true, HelpMessage = "Specifies the update Id we should import to WSUS", ParameterSetName = "Single")]
    # Specifies the update Id we should import to WSUS
    [ValidateNotNullOrEmpty()]
    [String]$UpdateId,

[Parameter(Mandatory = $true, HelpMessage = "Specifies path to a text file containing a list of update ID's on each line", ParameterSetName = "Multiple")]
    # Specifies path to a text file containing a list of update ID's on each line.
    [ValidateNotNullOrEmpty()]
    [String]$UpdateIdFilePath
)

Set-StrictMode -Version Latest

# set server options
$serverOptions = "Get-WsusServer"
if ($psBoundParameters.containsKey('WsusServer')) { $serverOptions += " -Name $WsusServer -PortNumber $PortNumber" }
if ($UseSsl) { $serverOptions += " -UseSsl" }

# empty updateID list
$updateList = @()

# get update id's
if ($UpdateIdFilePath) {
    if (Test-Path $UpdateIdFilePath) {
        foreach ($id in (Get-Content $UpdateIdFilePath)) {
            $updateList += $id.Trim()
        }
    }
    else {
        Write-Error "[$UpdateIdFilePath]: File not found"
		return
    }
}
else {
    $updateList = @($UpdateId)
}

# get WSUS server
Try {
    Write-Host "Attempting WSUS Connection using $serverOptions... " -NoNewline
    $server = invoke-expression $serverOptions
    Write-Host "Connection Successful"
}
Catch {
    Write-Error $_
    return
}

# empty file list
$FileList = @()

# call ImportUpdateFromCatalogSite on WSUS
foreach ($uid in $updateList) {
    Try {
        Write-Host "Attempting WSUS update import for Update ID: $uid... " -NoNewline
        $server.ImportUpdateFromCatalogSite($uid, $FileList)
        Write-Host "Import Successful"
    }
    Catch {
        Write-Error "Failed. $_"
    }
}
```

To check which updates are required for a Windows VM (Virtual Machine) using PowerShell, you can use the **`Get-WindowsUpdate`** cmdlet from the **PSWindowsUpdate module**, which helps to manage Windows updates. 

Here’s a step-by-step guide to check the required updates for the VM:

### Step 1: Install PSWindowsUpdate Module

If you don’t already have the **PSWindowsUpdate module** installed, you can install it from the PowerShell Gallery. Run the following command:

```powershell
Install-Module -Name PSWindowsUpdate -Force -Scope CurrentUser
```

### Step 2: Check for Pending Updates

Once the module is installed, you can check for the required updates using the `Get-WindowsUpdate` cmdlet.

```powershell
# Import the module (if not automatically imported)
Import-Module PSWindowsUpdate

# Get the list of pending updates (updates that are available but not yet installed)
$pendingUpdates = Get-WindowsUpdate

# Display the list of pending updates
$pendingUpdates | Format-Table -Property KBArticle, Title, Size, Installed
```

### Explanation:
- **`Get-WindowsUpdate`**: This cmdlet checks for updates available for the system. By default, it shows updates that have not been installed yet.
- **`Format-Table`**: This formats the output into a table with the following columns:
  - **KBArticle**: The KB article number for the update.
  - **Title**: A short description or title of the update.
  - **Size**: The size of the update.
  - **Installed**: Shows whether the update has been installed (`True` or `False`).

### Example Output:

| KBArticle | Title                           | Size    | Installed |
|-----------|---------------------------------|---------|-----------|
| KB5009557 | Cumulative Update for Windows 2022 | 500MB   | False     |
| KB5009876 | Security Update for Windows Server 2022 | 300MB   | False     |

### Step 3: Automatically Install Pending Updates

If you want to automatically install the pending updates after checking them, you can run the following command:

```powershell
# Automatically install all pending updates
Install-WindowsUpdate -AcceptAll -AutoReboot
```

- **`Install-WindowsUpdate`**: This cmdlet will install all available updates, and the **`-AutoReboot`** flag will ensure the VM reboots automatically if required by any of the updates.

### Step 4: Get Updates for a Specific KB (Optional)

If you're interested in checking if a specific update (KB) is required, you can filter the output:

```powershell
# Get pending updates for a specific KB article
$kbId = "KB5009557"
$pendingUpdates = Get-WindowsUpdate | Where-Object {$_.KBArticle -eq $kbId}
$pendingUpdates | Format-Table -Property KBArticle, Title, Size, Installed
```

This will show if the specific KB update is pending for installation.

### Step 5: List Installed Updates (Optional)

If you want to check what updates have already been installed on your VM (as opposed to the ones that are pending), you can use:

```powershell
# Get list of installed updates
Get-WindowsUpdate -IsInstalled | Format-Table -Property KBArticle, Title, Installed
```

### Example of Complete Script:

```powershell
# Install the PSWindowsUpdate module if not installed already
Install-Module -Name PSWindowsUpdate -Force -Scope CurrentUser

# Import the module
Import-Module PSWindowsUpdate

# Get pending updates
$pendingUpdates = Get-WindowsUpdate

# Display the list of pending updates
$pendingUpdates | Format-Table -Property KBArticle, Title, Size, Installed

# Optionally, automatically install the pending updates
# Install-WindowsUpdate -AcceptAll -AutoReboot
```

### Summary:
- Use **PSWindowsUpdate** module to check for required updates.
- **Get-WindowsUpdate** lists pending updates, and you can filter them or install them automatically.
- You can format the output to check the KB articles, update titles, and their installation status.

### Resource
Auto Update - https://www.starwindsoftware.com/blog/wsus-import-updates-new-powershell-import-method/

Troubleshooting - https://4sysops.com/archives/import-updates-manually-into-wsus-with-ie-or-powershell/

Manual - https://rdr-it.com/en/wsus-manually-import-an-update-from-the-microsoft-update-catalog
