# Active Directory Lab with Splunk SIEM

## Hardware Requirements

- **RAM:** 16GB (32 GB recommended) 
- **Storage:** 250GB+ free disk space (SSD strongly recommended for VM performance)
- **CPU:** 4+ cores with virtualization support (Intel VT-x or AMD-V must be enabled in BIOS)

## Software to Download

| Component                  | OS   | Download                                        |
| -------------------------- | ---- | ----------------------------------------------- |
| VirtualBox                 | Host | virtualbox.org                                  |
| Windows Server 2022        | VM   | Microsoft Official Iso                          |
| Windows 10/11              | VM   | Microsoft Official Iso                          |
| Ubuntu 24.04 LTS           | VM   | ubuntu.com                                      |
| Kali Linux                 | VM   | kali.org (pre-built VirtualBox image available) |
| Splunk Enterprise          | App  | splunk.com (free trial, 500MB/day ingest)       |
| Splunk Universal Forwarder | App  | splunk.com                                      |

## VirtualBox Network Setup

Before creating VMs, set up an isolated network in VirtualBox:

1. Open VirtualBox → **File → Tools → Network Manager**
2. Click **NAT Network** → **Create**
3. Set the IP prefix to `192.168.10.0/24` (or CTF-style `10.10.10.0/24`), subnet mask `255.255.255.0`.
4. Enable DHCP (Can assign IP manually later).

![[assets/Pasted image 20260310215152.png]]

- For each VM, go to Setting -> Network -> Attached to: NAT Network -> Choose the network created before.

## VM Specs

| VM                  | RAM     | CPU        | Disk     | OS                  |
| ------------------- | ------- | ---------- | -------- | ------------------- |
| Windows Server (DC) | 4GB     | 4 cores    | 60GB     | Windows Server 2022 |
| Windows Client      | 4GB     | 4 cores    | 50GB     | Windows 10/11       |
| ~~Ubuntu Client~~   | ~~2GB~~ | ~~1 core~~ | ~~30GB~~ | ~~Ubuntu 24.04~~    |
| Splunk Server       | 4GB     | 4 cores    | 120GB    | Ubuntu 24.04 Server |
| Kali Linux          | 2GB     | 2 cores    | 40GB     | Kali Linux          |

## Machine Setup
- [Windows Server Setup](Windows%20Server%20Setup.md)
- [Windows Client](./Windows%20Client)
- [Splunk-Ubuntu Setup](./Splunk-Ubuntu%20Setup)

Note: 
- Remember to sync time
- If log options of Splunk forwarder installation are ticked, then check `C:\Program Files\SplunkUniversalForwarder\etc\apps\SplunkUniversalForwarder\local\inputs.conf`. That yours place for inputs file. Careful with duplication.
## IP Assignment Table

| VM                  | Hostname        | Static IP       | Gateway        | DNS             |
| ------------------- | --------------- | --------------- | -------------- | --------------- |
| Splunk Server       | `splunk-server` | `192.168.10.10` | `192.168.10.1` | `8.8.8.8`       |
| Windows Server (DC) | `dc-server`     | `192.168.10.20` | `192.168.10.1` | `127.0.0.1`     |
| Windows Client      | `win-client`    | `192.168.10.21` | `192.168.10.1` | `192.168.10.20` |
| Kali Linux          | `kali-attacker` | `192.168.10.99` | `192.168.10.1` | `8.8.8.8`       |


# Labs

- [Labs 1 Brute-Force Detection with Splunk](Labs/Labs%201%20Brute-Force%20Detection%20with%20Splunk.md)