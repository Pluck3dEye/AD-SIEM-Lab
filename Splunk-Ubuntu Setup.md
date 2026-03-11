# Splunk Server (Ubuntu) Setup

## Initial Installation
### Partitioning

| Partition   | Mount Point       | Size | Filesystem | Purpose                                      |
| ----------- | ----------------- | ---- | ---------- | -------------------------------------------- |
| partition 1 | BIOS grub spacer  | 1MB  | —          | Required for BIOS boot, no filesystem needed |
| partition 2 | `/boot`           | 2GB  | ext4       | Boot files, kernel, GRUB                     |
| partition 3 | `swap`            | 2GB  | swap       | Virtual memory overflow                      |
| partition 4 | `/`               | 25GB | ext4       | OS root, system files                        |
| partition 5 | `/opt`            | 15GB | ext4       | Splunk application binaries                  |
| partition 6 | `/var`            | 5GB  | ext4       | System logs, temp files                      |
| partition 7 | `/opt/splunk/var` | 71GB | XFS        | Splunk indexes & all ingested log data       |

![[assets/Pasted image 20260310205438.png]]

## Default Account Information

| Field                     | What to Enter                  | Example         |
| ------------------------- | ------------------------------ | --------------- |
| **Your name**             | Your full name or just a label | `Splunk Admin`  |
| **Your server's name**    | Hostname for this machine      | `splunk-server` |
| **Pick a username**       | Short lowercase username       | `splunkadmin`   |
| **Choose a password**     | Strong password                | `MyPassword123` |
| **Confirm your password** | Same password again            | `MyPassword123` |

- After installation completes, update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

- Take a snapshot in VirtualBox before proceeding.


## Set Up Static IP

Identify your network adapter name:

```bash
ip a
```

In VirtualBox, the adapter is typically `enp0s3`.

Edit the Netplan configuration:

```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

```
network: 
	version: 2
	ethernets: 
		enp0s3: 
			dhcp4: no
			addresses: [192.168.10.10/24]
			nameservers: 
				addresses: [8.8.8.8]
			routes:
				- to: default
				  via: 192.168.10.1
```

After editing, secure the file and prevent cloud-init from overwriting it:

```bash
sudo chmod 600 /etc/netplan/00-installer-config.yaml
```

Disable cloud-init network management:

```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```

Add this new line:

```
network: {config: disabled}
```

Apply the configuration and verify:

```bash
sudo netplan apply
ip a
```


![[assets/Pasted image 20260310222818.png]]


## Install Splunk

```bash
wget -O splunk-10.2.1-c892b66d163d-linux-amd64.deb "https://download.splunk.com/products/splunk/releases/10.2.1/linux/splunk-10.2.1-c892b66d163d-linux-amd64.deb"
```

```bash
sudo dpkg -i splunk-10.2.1-c892b66d163d-linux-amd64.deb

splunkadmin@splunk-server:~/Downloads$ sudo dpkg -i splunk-10.2.1-c892b66d163d-linux-amd64.deb
Selecting previously unselected package splunk.
(Reading database ... 87770 files and directories currently installed.)
Preparing to unpack splunk-10.2.1-c892b66d163d-linux-amd64.deb ...
verify that this sytem has all the commands we will require to perform the preflight step
no need to run the splunk-preinstall upgrade check
Unpacking splunk (10.2.1) ...
Setting up splunk (10.2.1) ...

find: ‘/opt/splunk/lib/python3.7/site-packages’: No such file or directory
complete
```

(Ignore the warning)

Start Splunk
```bash
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root
```

```
splunkadmin@splunk-server:~/Downloads$ sudo /opt/splunk/bin/splunk start --accept-license --run-as-root

This appears to be your first time running this version of Splunk.

Splunk software must create an administrator account during startup. Otherwise, you cannot log in.
Create credentials for the administrator account.
Characters do not appear on the screen when you type in credentials.

Please enter an administrator username: admin
Password must contain at least:
   * 8 total printable ASCII character(s).
Please enter a new password:
Please confirm new password:
Copying '/opt/splunk/etc/openldap/ldap.conf.default' to '/opt/splunk/etc/openldap/ldap.conf'.
writing RSA key

writing RSA key

Moving '/opt/splunk/share/splunk/search_mrsparkle/modules.new' to '/opt/splunk/share/splunk/search_mrsparkle/modules'.

Splunk> All batbelt. No tights.

Checking prerequisites...
        Checking http port [8000]: open
        Checking mgmt port [8089]: open
        Checking appserver port [127.0.0.1:8065]: open
        Checking kvstore port [8191]: open
        Checking configuration... Done.
                Creating: /opt/splunk/var/lib/splunk
                Creating: /opt/splunk/var/run/splunk/appserver/i18n
                Creating: /opt/splunk/var/run/splunk/appserver/modules/static/css
                Creating: /opt/splunk/var/run/splunk/upload
                Creating: /opt/splunk/var/run/splunk/search_telemetry
                Creating: /opt/splunk/var/run/splunk/search_log
                Creating: /opt/splunk/var/spool/splunk
                Creating: /opt/splunk/var/spool/dirmoncache
                Creating: /opt/splunk/var/lib/splunk/authDb
                Creating: /opt/splunk/var/lib/splunk/hashDb
                Creating: /opt/splunk/var/run/splunk/collect
                Creating: /opt/splunk/var/run/splunk/sessions
New certs have been generated in '/opt/splunk/etc/auth'.
New certs have been generated in '/opt/splunk/etc/auth'.
        Checking critical directories...        Done
        Checking indexes...
                Validated: _audit _configtracker _dm_summary _dsappevent _dsclient _dsphonehome _internal _introspection _metrics _metrics_rollup _telemetry _thefishbucket history main summary
        Done
        Checking filesystem compatibility...  Done
        Checking conf files for problems...
        Done
        Checking default conf files for edits...
        Validating installed files against hashes from '/opt/splunk/splunk-10.2.1-c892b66d163d-linux-amd64-manifest'
        All installed files intact.
        Done
All preliminary checks passed.

Starting splunk server daemon (splunkd)...
Using configuration from /opt/splunk/share/openssl3/openssl.cnf
<SNIP>
Warning: ignoring -extensions option without -extfile
Certificate request self-signature ok
subject=CN = splunk-server, O = SplunkUser
Done


Waiting for web server at http://127.0.0.1:8000 to be available.........

```


| Field    | Value      |
| -------- | ---------- |
| Username | `admin`    |
| Password | `admin123` |

### Configure Splunk Service Account and Boot-start

Create a dedicated `splunk` service user and transfer ownership:

```bash
sudo useradd -m -r splunk
sudo chown -R splunk:splunk /opt/splunk
```

Enable boot-start using systemd (recommended for Ubuntu 24.04):

```bash
sudo /opt/splunk/bin/splunk enable boot-start -user splunk -systemd-managed 1
```

Reload and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable Splunkd
sudo systemctl start Splunkd
sudo systemctl status Splunkd
```


## Port Forwarding Rules

Configure port forwarding in VirtualBox to access the Splunk Server from the host machine.

### NAT Network Port Forwarding (Network Manager → AD-SIEM-NAT → Port Forwarding)

| Name        | Protocol | Host IP   | Host Port | Guest IP        | Guest Port |
| ----------- | -------- | --------- | --------- | --------------- | ---------- |
| SSH         | TCP      | 127.0.0.1 | 2222      | `192.168.10.10` | 22         |
| Splunk-Web  | TCP      | 127.0.0.1 | 8000      | `192.168.10.10` | 8000       |
| Splunk-Mgmt | TCP      | 127.0.0.1 | 8089      | `192.168.10.10` | 8089       |

## Access the Splunk Web UI

Open a browser on the host machine and navigate to `http://127.0.0.1:8000`.

![[assets/Pasted image 20260311103414.png]]


Log in with the credentials created earlier (`admin` / `admin123`).

![[assets/Pasted image 20260311103540.png]]


## Create Index and Configure Receiving Port

1. Navigate to **Settings → Indexes → New Index**.
2. Set **Index Name** to `windows` and **Index Type** to `Events`.
3. Click **Save**.

For the receiving port:

1. Navigate to **Settings → Forwarding and Receiving → Configure Receiving → New Receiving Port**.
2. Set **Port** to `9997`.
3. Click **Save**.

Once Windows machines are configured with forwarders, you should see data arriving in the `windows` index.

![[assets/Pasted image 20260311130238.png]]


![[assets/Pasted image 20260311135428.png]]