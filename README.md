# Enterprise Proxy Discovery & Kali Linux VM Configuration

---

# PART 1: INFORMATION GATHERING ON WINDOWS

---

## Step 1: Dump All Proxy Settings

Open CMD and run:

```cmd
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings"
```

Note down from output:
```
ProxyEnable    → 1 means proxy is active
ProxyServer    → proxy2.corp.com:8080
AutoConfigURL  → http://wpad.corp.com/proxy.pac
ProxyOverride  → localhost;127.*;10.*
```

---

## Step 2: Check System-Level Proxy

```cmd
netsh winhttp show proxy
```

Output:
```
Proxy Server: proxy2.corp.com:8080
Bypass List:  <local>
```

---

## Step 3: Resolve Proxy Hostname to IP

```cmd
nslookup proxy2.corp.com
```

Output:
```
Name:    proxy2.corp.com
Address: 10.10.1.5
```

Write down both — you need the IP if DNS fails inside the VM.

---

## Step 4: Get Full Network Details

```cmd
ipconfig /all
```

Note down:
```
IPv4 Address    : 10.10.0.25
Default Gateway : 10.10.0.1
DNS Servers     : 10.10.1.1
                  10.10.1.2
```

---

## Step 5: Get Domain Information

```cmd
echo %USERDNSDOMAIN%
```

```cmd
echo %LOGONSERVER%
```

```cmd
echo %USERDOMAIN%
```

Output example:
```
USERDNSDOMAIN  → CORP.COM
LOGONSERVER    → \\DC1
USERDOMAIN     → CORP
```

---

## Step 6: Find Domain Controller FQDN

```cmd
nslookup %LOGONSERVER:~2%
```

Or:

```cmd
nltest /dclist:corp.com
```

Output:
```
dc1.corp.com
dc2.corp.com
```

Write down the primary DC — needed for Kerberos KDC config.

---

## Step 7: Verify Proxy Port is Reachable

```cmd
telnet proxy2.corp.com 8080
```

If telnet not available:

```cmd
powershell Test-NetConnection -ComputerName proxy2.corp.com -Port 8080
```

Success output:
```
TcpTestSucceeded : True
```

---

## Step 8: Read the PAC File

Open in browser:
```
http://wpad.corp.com/proxy.pac
```

Or fetch via CMD:
```cmd
curl http://wpad.corp.com/proxy.pac
```

Look for:
```javascript
return "PROXY proxy2.corp.com:8080";
return "DIRECT";
```

Note every proxy address listed and every bypass rule.

---

## Step 9: Confirm Proxy Uses Kerberos

```cmd
curl -v -x http://proxy2.corp.com:8080 http://example.com
```

Look for in output:
```
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Negotiate
Proxy-Authenticate: Kerberos
```

`Negotiate` or `Kerberos` confirms Kerberos auth.

---

## Step 10: Export Corporate CA Certificate

```
Win+R → certmgr.msc → Enter
```
```
Trusted Root Certification Authorities → Certificates
```
```
Find your company root CA certificate
Right-click → All Tasks → Export
```
```
Next → Base-64 encoded X.509 (.CER) → Next
```
```
Save as: corp-ca.crt → Desktop
```

---

## Step 11: Master Reference Table

Fill this in completely before moving to Kali:

| Setting | Value |
|---|---|
| Proxy hostname | `proxy2.corp.com` |
| Proxy IP | `10.10.1.5` |
| Proxy port | `8080` |
| Auth type | `Kerberos / Negotiate` |
| Domain name | `CORP.COM` |
| Domain short name | `CORP` |
| Domain controller | `dc1.corp.com` |
| DNS server 1 | `10.10.1.1` |
| DNS server 2 | `10.10.1.2` |
| Default gateway | `10.10.0.1` |
| PAC file URL | `http://wpad.corp.com/proxy.pac` |
| Bypass list | `localhost;127.*;10.*` |
| Your AD username | `youruser` |
| Corp CA cert file | `corp-ca.crt` |

---
---

# PART 2: VIRTUALBOX CONFIGURATION

---

## Step 12: Set Network Adapter to NAT

```
Open VirtualBox
```
```
Select Kali VM → Click Settings
```
```
Network → Adapter 1
```
```
Attached to: NAT
```
```
Click Advanced
```
```
Promiscuous Mode: Allow All
```
```
Click OK
```

---

## Step 13: Configure Shared Folder for Certificate Transfer

```
VirtualBox → Settings → Shared Folders
```
```
Click the + icon on the right
```
```
Folder Path: C:\Users\YourUser\Desktop
Folder Name: shared
Check: Auto-mount
Check: Make Permanent
```
```
Click OK → OK
```

---

## Step 14: Boot Kali

```
Click Start on the Kali VM
```
```
Wait for full boot to desktop
```
```
Open a terminal
```

---
---

# PART 3: KALI LINUX — BASE NETWORK SETUP

---

## Step 15: Verify Network Interface Has IP

```bash
ip a
```

```bash
ip route show
```

Expected:
```
default via 10.0.2.2 dev eth0
10.0.2.0/24 dev eth0 proto kernel
```

If no IP:

```bash
sudo dhclient eth0
```

---

## Step 16: Restart Network Services

```bash
sudo systemctl restart NetworkManager
```

```bash
sudo systemctl restart networking
```

---

## Step 17: Test VirtualBox Gateway

```bash
ping -c 3 10.0.2.2
```

Must succeed before going further — this confirms NAT is working.

---

## Step 18: Configure DNS

```bash
sudo nano /etc/resolv.conf
```

Add:
```
nameserver 10.10.1.1
nameserver 10.10.1.2
nameserver 8.8.8.8
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Lock it from being overwritten:

```bash
sudo chattr +i /etc/resolv.conf
```

Verify lock:

```bash
lsattr /etc/resolv.conf
```

Output:
```
----i---------e--- /etc/resolv.conf
```

Test DNS:

```bash
nslookup google.com
```

```bash
nslookup dc1.corp.com
```

Both must resolve before continuing.

---

## Step 19: Import Corporate CA Certificate

Mount the shared folder:

```bash
sudo mkdir -p /mnt/shared
```

```bash
sudo mount -t vboxsf shared /mnt/shared
```

Copy and install:

```bash
sudo cp /mnt/shared/corp-ca.crt /usr/local/share/ca-certificates/
```

```bash
sudo update-ca-certificates
```

Verify:

```bash
ls /usr/local/share/ca-certificates/ | grep corp
```

```bash
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt /usr/local/share/ca-certificates/corp-ca.crt
```

---
---

# PART 4: KERBEROS CONFIGURATION

---

## Step 20: Install Kerberos Packages

```bash
sudo apt update
```

```bash
sudo apt install -y krb5-user cntlm curl libgssapi-krb5-2 ntpdate
```

During installation a blue dialog will appear asking for:

```
Default Kerberos version 5 realm:
→ Type: CORP.COM (use your actual domain in ALL CAPS)
```

```
Kerberos servers for your realm:
→ Type: dc1.corp.com
```

```
Administrative server for your Kerberos realm:
→ Type: dc1.corp.com
```

---

## Step 21: Configure Kerberos

```bash
sudo nano /etc/krb5.conf
```

Replace entire contents with:

```ini
[libdefaults]
    default_realm = CORP.COM
    dns_lookup_realm = false
    dns_lookup_kdc = true
    ticket_lifetime = 24h
    renew_lifetime = 7d
    forwardable = true
    rdns = false

[realms]
    CORP.COM = {
        kdc = dc1.corp.com
        admin_server = dc1.corp.com
        default_domain = corp.com
    }

[domain_realm]
    .corp.com = CORP.COM
    corp.com = CORP.COM

[logging]
    default = FILE:/var/log/krb5libs.log
    kdc = FILE:/var/log/krb5kdc.log
    admin_server = FILE:/var/log/kadmind.log
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

---

## Step 22: Sync Time with Domain Controller

Kerberos fails if clock is off by more than 5 minutes:

```bash
sudo ntpdate dc1.corp.com
```

Set correct timezone:

```bash
sudo timedatectl set-timezone Africa/Nairobi
```

Enable NTP sync:

```bash
sudo timedatectl set-ntp true
```

Verify time matches Windows host:

```bash
date
```

---

## Step 23: Test Kerberos Connectivity to KDC

```bash
ping -c 3 dc1.corp.com
```

```bash
nmap -p 88 dc1.corp.com
```

Port 88 must be open — that is the Kerberos port.

---

## Step 24: Get a Kerberos Ticket

```bash
kinit youruser@CORP.COM
```

Enter your Windows domain password when prompted.

Verify ticket was issued:

```bash
klist
```

Expected output:
```
Ticket cache: FILE:/tmp/krb5cc_1000
Default principal: youruser@CORP.COM

Valid starting       Expires              Service principal
03/25/2026 08:00:00  03/25/2026 18:00:00  krbtgt/CORP.COM@CORP.COM
        renew until 04/01/2026 08:00:00
```

---

## Step 25: Test Kerberos Against Proxy Directly

```bash
curl -v --proxy-negotiate -u : -x http://proxy2.corp.com:8080 https://google.com
```

If you see `HTTP/1.1 200 OK` — Kerberos auth works directly.

If you see errors — proceed with Cntlm bridge in next steps.

---
---

# PART 5: CNTLM CONFIGURATION (KERBEROS BRIDGE)

---

## Step 26: Configure Cntlm

```bash
sudo nano /etc/cntlm.conf
```

Replace contents with:

```
Username        youruser
Domain          CORP
Proxy           proxy2.corp.com:8080
NoProxy         localhost, 127.0.0.1, 10.0.0.0/8, 192.168.0.0/16
Listen          3128
Auth            GSS
GSSApiFlags     +DELEGATION
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

---

## Step 27: Start and Enable Cntlm

```bash
sudo systemctl enable cntlm
```

```bash
sudo systemctl start cntlm
```

Check status:

```bash
sudo systemctl status cntlm
```

Output should show:
```
Active: active (running)
```

---

## Step 28: Test Cntlm is Working

```bash
curl -x http://localhost:3128 -I https://google.com
```

Expected:
```
HTTP/1.1 200 OK
```

---
---

# PART 6: CONFIGURE ALL TOOLS TO USE CNTLM

---

## Step 29: System-Wide Proxy

```bash
sudo nano /etc/environment
```

Add:

```
http_proxy="http://localhost:3128"
https_proxy="http://localhost:3128"
ftp_proxy="http://localhost:3128"
HTTP_PROXY="http://localhost:3128"
HTTPS_PROXY="http://localhost:3128"
FTP_PROXY="http://localhost:3128"
no_proxy="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
NO_PROXY="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Reload:

```bash
source /etc/environment
```

Verify:

```bash
printenv | grep -i proxy
```

---

## Step 30: apt Proxy

```bash
sudo nano /etc/apt/apt.conf.d/99proxy
```

Add:

```
Acquire::http::Proxy "http://localhost:3128";
Acquire::https::Proxy "http://localhost:3128";
Acquire::ftp::Proxy "http://localhost:3128";
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Test:

```bash
sudo apt update
```

---

## Step 31: curl Proxy

```bash
nano ~/.curlrc
```

Add:

```
proxy = "http://localhost:3128"
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Test:

```bash
curl -I https://google.com
```

---

## Step 32: wget Proxy

```bash
nano ~/.wgetrc
```

Add:

```
http_proxy = http://localhost:3128
https_proxy = http://localhost:3128
ftp_proxy = http://localhost:3128
use_proxy = on
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Test:

```bash
wget -q --spider https://google.com && echo "wget OK"
```

---

## Step 33: git Proxy

```bash
git config --global http.proxy http://localhost:3128
```

```bash
git config --global https.proxy http://localhost:3128
```

```bash
git config --global http.sslVerify true
```

Verify:

```bash
git config --global --list | grep proxy
```

Test:

```bash
git ls-remote https://github.com/git/git HEAD
```

---

## Step 34: pip Proxy

```bash
mkdir -p ~/.config/pip
```

```bash
nano ~/.config/pip/pip.conf
```

Add:

```
[global]
proxy = http://localhost:3128
trusted-host = pypi.org
               pypi.python.org
               files.pythonhosted.org
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Test:

```bash
pip install requests --dry-run
```

---

## Step 35: snap Proxy

```bash
sudo snap set system proxy.http="http://localhost:3128"
```

```bash
sudo snap set system proxy.https="http://localhost:3128"
```

Verify:

```bash
sudo snap get system proxy
```

---

## Step 36: Root User Proxy

```bash
sudo nano /root/.bashrc
```

Add at bottom:

```bash
export http_proxy="http://localhost:3128"
export https_proxy="http://localhost:3128"
export HTTP_PROXY="http://localhost:3128"
export HTTPS_PROXY="http://localhost:3128"
export no_proxy="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
export NO_PROXY="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Preserve proxy vars when using sudo:

```bash
sudo visudo
```

Find:
```
Defaults env_reset
```

Add directly below:

```
Defaults env_keep += "http_proxy https_proxy ftp_proxy HTTP_PROXY HTTPS_PROXY FTP_PROXY no_proxy NO_PROXY"
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

---
---

# PART 7: TICKET AUTO-RENEWAL

---

## Step 37: Create Auto-Renewal Cron Job

```bash
sudo nano /etc/cron.hourly/krb5-renew
```

Add:

```bash
#!/bin/bash
export KRB5CCNAME=/tmp/krb5cc_$(id -u)
kinit -R 2>/dev/null
if [ $? -ne 0 ]; then
    echo "Kerberos ticket renewal failed at $(date)" >> /var/log/krb5-renew.log
fi
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Make executable:

```bash
sudo chmod +x /etc/cron.hourly/krb5-renew
```

Manually renew ticket anytime:

```bash
kinit -R
```

Check ticket status and expiry:

```bash
klist
```

---
---

# PART 8: FINAL VERIFICATION

---

## Step 38: Run Full Diagnostic

```bash
ip a
```

```bash
ip route show
```

```bash
cat /etc/resolv.conf
```

```bash
printenv | grep -i proxy
```

```bash
ping -c 3 10.0.2.2
```

```bash
ping -c 3 dc1.corp.com
```

```bash
nslookup google.com
```

```bash
klist
```

```bash
curl -I https://google.com
```

```bash
wget -q --spider https://google.com && echo "wget OK"
```

```bash
sudo apt update
```

```bash
git ls-remote https://github.com/git/git HEAD
```

```bash
pip install requests --dry-run
```

---

## Step 39: Save Diagnostic Script

```bash
sudo nano /usr/local/bin/netcheck.sh
```

Paste:

```bash
#!/bin/bash
echo "================================================"
echo "         FULL NETWORK DIAGNOSTIC REPORT"
echo "================================================"

echo ""
echo "[1] NETWORK INTERFACE"
ip a | grep -E "inet|state|eth|enp"

echo ""
echo "[2] ROUTING TABLE"
ip route show

echo ""
echo "[3] DNS SERVERS"
cat /etc/resolv.conf | grep nameserver

echo ""
echo "[4] PROXY ENVIRONMENT VARIABLES"
printenv | grep -i proxy

echo ""
echo "[5] GATEWAY REACHABILITY"
ping -c 2 10.0.2.2 | tail -2

echo ""
echo "[6] DOMAIN CONTROLLER REACHABILITY"
ping -c 2 dc1.corp.com | tail -2

echo ""
echo "[7] DNS RESOLUTION"
nslookup google.com | tail -3

echo ""
echo "[8] KERBEROS TICKET STATUS"
klist 2>/dev/null || echo "No Kerberos ticket found - run: kinit user@CORP.COM"

echo ""
echo "[9] CNTLM STATUS"
sudo systemctl status cntlm | grep -E "Active|running|failed"

echo ""
echo "[10] HTTP CONNECTIVITY VIA CNTLM"
curl -sx http://localhost:3128 -o /dev/null -w "HTTP Status: %{http_code}\n" https://google.com

echo ""
echo "[11] APT PROXY CONFIG"
cat /etc/apt/apt.conf.d/99proxy

echo ""
echo "[12] CERTIFICATE STORE"
ls /usr/local/share/ca-certificates/ | grep corp

echo ""
echo "================================================"
echo "              END OF REPORT"
echo "================================================"
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

```bash
sudo chmod +x /usr/local/bin/netcheck.sh
```

Run anytime:

```bash
sudo netcheck.sh
```

---

# MASTER CHECKLIST

| # | Task | Status |
|---|---|---|
| 1 | Proxy address + port from reg query | ☐ |
| 2 | Proxy IP from nslookup | ☐ |
| 3 | Kerberos auth confirmed via 407 Negotiate | ☐ |
| 4 | Domain name + DC from echo %USERDNSDOMAIN% | ☐ |
| 5 | DNS servers from ipconfig /all | ☐ |
| 6 | PAC file read | ☐ |
| 7 | Corp CA cert exported from certmgr.msc | ☐ |
| 8 | VirtualBox adapter set to NAT | ☐ |
| 9 | Shared folder configured | ☐ |
| 10 | Kali has IP 10.0.2.x | ☐ |
| 11 | Gateway ping 10.0.2.2 works | ☐ |
| 12 | DNS locked in /etc/resolv.conf | ☐ |
| 13 | DC reachable from Kali | ☐ |
| 14 | Corp CA cert imported | ☐ |
| 15 | krb5-user + cntlm installed | ☐ |
| 16 | /etc/krb5.conf configured | ☐ |
| 17 | Time synced with DC via ntpdate | ☐ |
| 18 | kinit succeeds and klist shows ticket | ☐ |
| 19 | Cntlm running on localhost:3128 | ☐ |
| 20 | /etc/environment points to localhost:3128 | ☐ |
| 21 | apt, curl, wget, git, pip all configured | ☐ |
| 22 | Root user proxy set | ☐ |
| 23 | sudo preserves proxy env vars | ☐ |
| 24 | Kerberos ticket auto-renewal cron set | ☐ |
| 25 | netcheck.sh passes all checks | ☐ |

---

> Replace every instance of `CORP.COM`, `dc1.corp.com`, `proxy2.corp.com:8080`, and `youruser` with the exact values collected in Part 1. Those four values drive the entire configuration.
