# learn-wsus
how to wsus

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
