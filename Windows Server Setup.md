# Windows Server (Domain Controller) Setup

## Installation

- Choose `Windows Server Datacenter (Desktop Experience) - 10/9/2025`.
- Accept the license terms, then choose **Custom Installation**.
- Create new partitions in unallocated space.
- Install on the primary partition.
- Restart.
- Create a password for the Administrator account (e.g., `4dminP4ss123@`).
- Use `Host + Del` in VirtualBox to unlock the login screen.

## Rename the Server

Rename the PC to `DC01`, then restart:

## Set Static IP

Open PowerShell as Administrator and configure the static IP:

```powershell
# Remove existing DHCP configuration
Remove-NetIPAddress -InterfaceAlias "Ethernet" -Confirm:$false
Remove-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix "0.0.0.0/0" -Confirm:$false

# Assign static IP
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "192.168.10.20" -PrefixLength 24 -DefaultGateway "192.168.10.1"
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "8.8.8.8","8.8.4.4"
```

> **Note:** After promoting this server to a Domain Controller, the DNS will automatically point to `127.0.0.1` (itself).

## Install Sysmon

Download Sysmon and the Olaf Hartong configuration file (see [[Windows Client#Install Sysmon]] for download links). Place both in the same directory and run:

```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

## Install Splunk Universal Forwarder

Install the Splunk Universal Forwarder with the Receiving Indexer set to `192.168.10.10:9997` (see [[Windows Client#Install Splunk Universal Forwarder]] for detailed steps).

Navigate to the local configuration directory and create/edit the configuration files:

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\
```

### inputs.conf

This configuration collects the same logs as the Windows Client, plus DC-specific event logs (Directory Service, DNS Server, Group Policy):

```ini
[WinEventLog://Application]
index = windows
disabled = false
renderXml = true
source = XmlWinEventLog:Application

[WinEventLog://Security]
index = windows
disabled = false
renderXml = true
source = XmlWinEventLog:Security

[WinEventLog://System]
index = windows
disabled = false
renderXml = true
source = XmlWinEventLog:System

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = windows
disabled = false
renderXml = true
source = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational

[WinEventLog://Microsoft-Windows-PowerShell/Operational]
index = windows
disabled = false
renderXml = true
source = XmlWinEventLog:Microsoft-Windows-PowerShell/Operational

[WinEventLog://Microsoft-Windows-WMI-Activity/Operational]
index = windows
disabled = false
renderXml = true
source = XmlWinEventLog:Microsoft-Windows-WMI-Activity/Operational

#DC specific
[WinEventLog://Directory Service]
index = windows
disabled = false
renderXml = true
source = XmlWinEventLog:Directory-Service

[WinEventLog://DNS Server]
index = windows
disabled = false
renderXml = true
source = XmlWinEventLog:DNS-Server

[WinEventLog://Microsoft-Windows-GroupPolicy/Operational]
index = windows
disabled = false
renderXml = true
source = XmlWinEventLog:Microsoft-Windows-GroupPolicy/Operational
```

### outputs.conf

```ini
[tcpout]
defaultGroup = splunk-server

[tcpout:splunk-server]
server = 192.168.10.10:9997
compressed = true

[tcpout-server://192.168.10.10:9997]
```

Restart the forwarder service to apply the changes:

```powershell
Restart-Service SplunkForwarder
```

## Active Directory Domain Services (ADDS) Setup

1. Open **Server Manager** → **Add Roles and Features**.
2. **Installation Type:** Select **Role-based or feature-based installation**.
![[assets/Pasted image 20260311152735.png]]

3. **Server Selection:** Choose `DC01`.

![[assets/Pasted image 20260311152710.png]]

4. Select the **Active Directory Domain Services** role and add required features.
5. Click **Next** through the remaining pages and **Install**.

After the role is installed, click **Promote this server to a domain controller**.

![[assets/Pasted image 20260311153548.png]]



### Domain Controller Promotion

- **Deployment Configuration:** Add a new forest with root domain name `corp.local`.
- **DSRM Password:** `DSRMPass123`
- **NetBIOS Name:** Accept the default (`CORP`).
- Click **Next** through the remaining pages and **Install**.
- The server will restart automatically after promotion.

## Create Organizational Units and Users

Create the top-level OUs:

```powershell
New-ADOrganizationalUnit -Name "IT" -Path "DC=corp,DC=local" 
New-ADOrganizationalUnit -Name "HR" -Path "DC=corp,DC=local"
```

Create sub-OUs for Users and Computers within each department:

```powershell
New-ADOrganizationalUnit -Name "Users" -Path "OU=IT,DC=corp,DC=local" 
New-ADOrganizationalUnit -Name "Computers" -Path "OU=IT,DC=corp,DC=local" 
New-ADOrganizationalUnit -Name "Users" -Path "OU=HR,DC=corp,DC=local" 
New-ADOrganizationalUnit -Name "Computers" -Path "OU=HR,DC=corp,DC=local"
```

Verify the OU structure:

```powershell
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName
```

![[assets/Pasted image 20260311171456.png]]


### Create Users

```powershell
New-ADUser -Name "John Smith" `
-GivenName "John" -Surname "Smith" `
-SamAccountName "jsmith" `
-UserPrincipalName "jsmith@corp.local" `
-Path "OU=Users,OU=IT,DC=corp,DC=local" `
-AccountPassword (ConvertTo-SecureString "JohnPassword123!" -AsPlainText -Force) `
-Enabled $true

New-ADUser -Name "Jane Doe" `
-GivenName "Jane" -Surname "Doe" `
-SamAccountName "jdoe" `
-UserPrincipalName "jdoe@corp.local" `
-Path "OU=Users,OU=HR,DC=corp,DC=local" `
-AccountPassword (ConvertTo-SecureString "JanePassword123!" -AsPlainText -Force) `
-Enabled $true

```

Verify the users were created:

```powershell
Get-ADUser -Filter * | Select-Object Name, SamAccountName, Enabled
```