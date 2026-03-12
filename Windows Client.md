# Windows Client Setup

## Installation

- Choose `Windows 10 Enterprise 10/11/2025`.
- Accept the license terms, then choose **Custom Installation**.
- Create new partitions in unallocated space.
- Install on the primary partition.
- Restart.

Credential: `Bob` / `BobPassword123`

Rename the PC to `WS01` (or your preferred hostname), then restart.
## Set Static IP

Open PowerShell as Administrator:

```powershell
Get-NetAdapter
```

Remove the existing DHCP address and default route:

```powershell
Remove-NetIPAddress -InterfaceAlias "Ethernet" -Confirm:$false
Remove-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix "0.0.0.0/0" -Confirm:$false
```

Set the static IP:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress "192.168.10.21" -PrefixLength 24 -DefaultGateway "192.168.10.1"
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "8.8.8.8","8.8.4.4"
```

> **Note:** After setting up Active Directory on the Windows Server, change the DNS server to the DC's IP (`192.168.10.20`) so the client can resolve the domain.

To verify connectivity with the Splunk server, try pinging or opening `http://192.168.10.10:8000` in a browser.


## Install Splunk Universal Forwarder

Download the installer:

```powershell
wget -O splunkforwarder-10.2.1-c892b66d163d-windows-x64.msi "https://download.splunk.com/products/universalforwarder/releases/10.2.1/windows/splunkforwarder-10.2.1-c892b66d163d-windows-x64.msi"
```

During installation:

- Accept the license and choose **An on-premises Splunk Enterprise Instance**.
- Leave **SSL Certificate** and **SSL Root CA** fields blank and click **Next**.
- Select **Install as Local System**.
- LEAVE LOGS SELECTION BLANK and click next
- Credential: `admin` / `<your-generated-password>`
- **Deployment Server:** Leave blank.
- **Receiving Indexer:**
	- Hostname or IP: `192.168.10.10`
	- Port: `9997`

## Install Sysmon

Download the required files:

- **Sysmon:** https://download.sysinternals.com/files/Sysmon.zip
- **Sysmon Config (Olaf Hartong — recommended):** https://github.com/olafhartong/sysmon-modular/blob/master/sysmonconfig.xml

Alternative configurations:
- **SwiftOnSecurity:** https://github.com/SwiftOnSecurity/sysmon-config
- **Neo23x0 (Florian Roth):** https://github.com/NextronSystems/sysmon-config

Extract Sysmon and place the config file in the same directory. Open PowerShell as Administrator and run:

```powershell
.\Sysmon64.exe -accepteula -i sysmonconfig.xml
```

Verify Sysmon is running:

```powershell
Get-Service Sysmon64
```

## Configure Forwarder

### Configuration Files

The Splunk Universal Forwarder uses two key configuration files:

**1. `inputs.conf`** — defines **what logs to collect**

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf
```

**2. `outputs.conf`** — defines **where to send logs**

```
C:\Program Files\SplunkUniversalForwarder\etc\system\local\outputs.conf
```

### inputs.conf

```ini
[WinEventLog://Application]
index = windows
renderXml = true
start_from = oldest
checkpointInterval = 5

[WinEventLog://Security]
index = windows
renderXml = true
start_from = oldest
checkpointInterval = 5

[WinEventLog://System]
index = windows
renderXml = true
start_from = oldest
checkpointInterval = 5

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = windows
renderXml = true
start_from = oldest
checkpointInterval = 5

[WinEventLog://Microsoft-Windows-PowerShell/Operational]
index = windows
renderXml = true
start_from = oldest
checkpointInterval = 5

[WinEventLog://Microsoft-Windows-WMI-Activity/Operational]
index = windows
renderXml = true
start_from = oldest
checkpointInterval = 5


# Domain Controller logs

[WinEventLog://Directory Service]
index = windows
renderXml = true
start_from = oldest
checkpointInterval = 5

[WinEventLog://DNS Server]
index = windows
renderXml = true
start_from = oldest
checkpointInterval = 5

[WinEventLog://Microsoft-Windows-GroupPolicy/Operational]
index = windows
renderXml = true
start_from = oldest
checkpointInterval = 5
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

### Verify Forwarder Service Account

The forwarder must run as **LocalSystem** to collect Security logs. If it runs under a different account, Security logs will silently fail to appear in Splunk.

Check the current service account:

```powershell
Get-WmiObject Win32_Service -Filter "Name='SplunkForwarder'" | Select-Object StartName
```

If not running as LocalSystem, fix it:
```powershell
sc.exe config SplunkForwarder obj="LocalSystem"
```

Restart the forwarder:
```powershell
Stop-Service SplunkForwarder -Force 
Start-Service SplunkForwarder
```

After restarting, verify data is arriving in Splunk by navigating to the Splunk Web UI and checking the `windows` index. See [[Splunk-Ubuntu Setup#Create Index and Configure Receiving Port]].

![[assets/Pasted image 20260311130238.png]]


## Join Domain

After Active Directory Domain Services (ADDS) is configured on the Windows Server, join this client to the domain.

**Prerequisites:** Ensure the DNS server is set to the Windows Server IP (`192.168.10.20`).

```powershell
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses "192.168.10.20"
```

Join the domain (interactive password prompt):

```
Add-Computer -DomainName "corp.local" -Credential corp\Administrator -OUPath "OU=Computers,OU=IT,DC=corp,DC=local" -Restart
```

Then type the Administrator password when prompted.

Alternatively, join with credentials inline (scripted approach):
```powershell
$password = ConvertTo-SecureString "4dminP4ss123@" -AsPlainText -Force 
$credential = New-Object System.Management.Automation.PSCredential("corp\Administrator", $password) 
Add-Computer -DomainName "corp.local" ` -Credential $credential ` -OUPath "OU=Computers,OU=IT,DC=corp,DC=local" ` -Restart
```

> **Note:** The `-OUPath` above places the computer in `OU=Computers,OU=IT`. For the default Computers container, use `"CN=Computers,DC=corp,DC=local"`. To move a computer after joining, run on the DC:
```powershell
Get-ADComputer -Identity "WS01" | Move-ADObject -TargetPath "OU=Computers,OU=HR,DC=corp,DC=local"
```

After the machine restarts, select **Other user** on the login screen and log in with a domain account (e.g., `jsmith` / `JohnPassword123!`).

![[assets/Pasted image 20260311174026.png]]

