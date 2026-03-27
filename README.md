# Enterprise Proxy Discovery & Kali Linux VM Configuration

Good catch. The certificate steps are there but **scattered and incomplete**. Specifically:

- Step 19 covers importing the cert — but it assumes the shared folder is already working
- There is **no step verifying the cert was actually trusted** by curl, wget, git, pip individually
- The shared folder mount is **not persistent** — it dies on reboot
- No step shows how to **transfer the cert if shared folders fail**

---

# Complete End-to-End Guide Including Certificate Installation

---

# PART 1: INFORMATION GATHERING ON WINDOWS

---

## Step 1: Dump All Proxy Settings

```cmd
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings"
```

Note down:
```
ProxyEnable    → 1
ProxyServer    → proxy2.corp.com:8080
AutoConfigURL  → http://wpad.corp.com/proxy.pac
ProxyOverride  → localhost;127.*;10.*
```

---

## Step 2: Check System-Level Proxy

```cmd
netsh winhttp show proxy
```

Note down:
```
Proxy Server: proxy2.corp.com:8080
Bypass List:  <local>
```

---

## Step 3: Resolve Proxy Hostname to IP

```cmd
nslookup proxy2.corp.com
```

Note down both hostname and IP:
```
Name:    proxy2.corp.com
Address: 10.10.1.5
```

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

Note down:
```
USERDNSDOMAIN → CORP.COM
LOGONSERVER   → \\DC1
USERDOMAIN    → CORP
```

---

## Step 6: Find Domain Controller FQDN

```cmd
nltest /dclist:corp.com
```

Note down primary DC:
```
dc1.corp.com
```

---

## Step 7: Verify Proxy Port is Reachable

```cmd
powershell Test-NetConnection -ComputerName proxy2.corp.com -Port 8080
```

Must show:
```
TcpTestSucceeded : True
```

---

## Step 8: Read the PAC File

Open in browser:
```
http://wpad.corp.com/proxy.pac
```

Note every proxy address and bypass rule inside it.

---

## Step 9: Confirm Proxy Uses Kerberos

```cmd
curl -v -x http://proxy2.corp.com:8080 http://example.com
```

Look for:
```
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: Negotiate
```

---

## Step 10: Export Corporate CA Certificate

```
Win+R → certmgr.msc → Enter
```
```
Trusted Root Certification Authorities → Certificates
```
```
Find your company root CA → Right-click → All Tasks → Export
```
```
Next → Base-64 encoded X.509 (.CER) → Next
```
```
Filename: corp-ca.crt → Save to Desktop
```

---

## Step 11: Also Export as DER Format (Needed for Java tools)

Repeat Step 10 but choose:
```
DER encoded binary X.509 (.CER)
Filename: corp-ca.der → Save to Desktop
```

---

## Step 12: Master Reference Table

| Setting | Your Value |
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
| Your AD username | `youruser` |
| Corp CA cert (Base64) | `corp-ca.crt` |
| Corp CA cert (DER) | `corp-ca.der` |

---
---

# PART 2: VIRTUALBOX CONFIGURATION

---

## Step 13: Set Network Adapter to NAT

```
VirtualBox → Select Kali VM → Settings
```
```
Network → Adapter 1
```
```
Attached to: NAT
```
```
Advanced → Promiscuous Mode: Allow All
```
```
Click OK
```

---

## Step 14: Configure Shared Folder

```
VirtualBox → Settings → Shared Folders → Click +
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

## Step 15: Boot Kali

```
Click Start → Wait for full boot → Open terminal
```

---
---

# PART 3: BASE NETWORK SETUP

---

## Step 16: Verify Network Interface

```bash
ip a
```

```bash
ip route show
```

If no IP:

```bash
sudo dhclient eth0
```

---

## Step 17: Restart Network Services

```bash
sudo systemctl restart NetworkManager
```

```bash
sudo systemctl restart networking
```

---

## Step 18: Test Gateway

```bash
ping -c 3 10.0.2.2
```

Must succeed before continuing.

---

## Step 19: Configure DNS

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

Lock from being overwritten:

```bash
sudo chattr +i /etc/resolv.conf
```

Verify lock:

```bash
lsattr /etc/resolv.conf
```

Test:

```bash
nslookup google.com
```

```bash
nslookup dc1.corp.com
```

---
---

# PART 4: CERTIFICATE INSTALLATION (COMPLETE)

This is the most critical part — every tool that uses HTTPS will fail without this.

---

## Step 20: Mount the Shared Folder

```bash
sudo mkdir -p /mnt/shared
```

```bash
sudo mount -t vboxsf shared /mnt/shared
```

Verify the cert is visible:

```bash
ls /mnt/shared/
```

You should see `corp-ca.crt` and `corp-ca.der` listed.

---

## Step 21: Make the Mount Persistent Across Reboots

```bash
sudo nano /etc/fstab
```

Add at the bottom:

```
shared  /mnt/shared  vboxsf  defaults,uid=1000,gid=1000,umask=0022  0  0
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

---

## Step 22: Copy Certificates to Correct Locations

```bash
sudo cp /mnt/shared/corp-ca.crt /usr/local/share/ca-certificates/corp-ca.crt
```

```bash
sudo cp /mnt/shared/corp-ca.crt /etc/ssl/certs/corp-ca.crt
```

```bash
sudo cp /mnt/shared/corp-ca.der /etc/ssl/certs/corp-ca.der
```

---

## Step 23: Install Certificate into System Trust Store

```bash
sudo update-ca-certificates
```

Expected output:
```
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
```

Verify it was added:

```bash
ls /usr/local/share/ca-certificates/ | grep corp
```

```bash
ls /etc/ssl/certs/ | grep corp
```

---

## Step 24: Verify Certificate is Valid

```bash
openssl x509 -in /usr/local/share/ca-certificates/corp-ca.crt -text -noout | grep -E "Issuer|Subject|Not Before|Not After"
```

```bash
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt /usr/local/share/ca-certificates/corp-ca.crt
```

Expected:
```
corp-ca.crt: OK
```

---

## Step 25: Install Certificate for curl

curl uses the system store automatically after `update-ca-certificates` but verify:

```bash
curl --cacert /etc/ssl/certs/ca-certificates.crt -I https://proxy2.corp.com
```

```bash
curl -I https://google.com
```

No SSL errors = cert is working for curl.

---

## Step 26: Install Certificate for wget

wget also uses system store — verify:

```bash
wget -q --spider https://google.com && echo "wget cert OK"
```

If still failing force the cert:

```bash
echo "ca_certificate=/etc/ssl/certs/ca-certificates.crt" >> ~/.wgetrc
```

---

## Step 27: Install Certificate for git

```bash
git config --global http.sslCAInfo /etc/ssl/certs/ca-certificates.crt
```

Verify:

```bash
git config --global --list | grep ssl
```

---

## Step 28: Install Certificate for pip

```bash
mkdir -p ~/.config/pip
```
```bash
nano ~/.config/pip/pip.conf
```

Add under `[global]`:

```
cert = /etc/ssl/certs/ca-certificates.crt
```

Full file should look like:
```
[global]
proxy = http://localhost:3128
cert = /etc/ssl/certs/ca-certificates.crt
trusted-host = pypi.org
               pypi.python.org
               files.pythonhosted.org
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

---

## Step 29: Install Certificate for Java (Burp, ZAP, etc.)

```bash
sudo keytool -import -trustcacerts \
  -alias corp-ca \
  -file /mnt/shared/corp-ca.der \
  -keystore /etc/ssl/certs/java/cacerts \
  -storepass changeit \
  -noprompt
```

Verify Java trust store:

```bash
sudo keytool -list -keystore /etc/ssl/certs/java/cacerts \
  -storepass changeit | grep corp-ca
```

---

## Step 30: Install Certificate for Firefox (if used)

```bash
sudo apt install libnss3-tools
```

```bash
certutil -A -n "Corp CA" \
  -t "CT,," \
  -i /mnt/shared/corp-ca.crt \
  -d ~/.mozilla/firefox/*.default-esr
```

Verify:

```bash
certutil -L -d ~/.mozilla/firefox/*.default-esr | grep Corp
```

---

## Step 31: Install Certificate for Chromium (if used)

```bash
mkdir -p ~/.pki/nssdb
```

```bash
certutil -d ~/.pki/nssdb -N --empty-password
```

```bash
certutil -d ~/.pki/nssdb -A \
  -n "Corp CA" \
  -t "CT,," \
  -i /mnt/shared/corp-ca.crt
```

Verify:

```bash
certutil -d ~/.pki/nssdb -L | grep Corp
```

---

## Step 32: Fallback — Transfer Cert Without Shared Folder

If shared folder mount fails use one of these alternatives:

**Option A — Python HTTP server on Windows:**

On Windows CMD in Desktop folder:
```cmd
python -m http.server 8000
```

On Kali:
```bash
wget http://10.10.0.25:8000/corp-ca.crt
wget http://10.10.0.25:8000/corp-ca.der
```

**Option B — Base64 paste directly into Kali terminal:**

On Windows CMD:
```cmd
certutil -encode corp-ca.crt corp-ca-b64.txt
```

Open `corp-ca-b64.txt` → copy contents → paste into Kali:
```bash
nano /tmp/corp-ca.crt
# paste contents
# Ctrl+O → Enter → Ctrl+X
sudo cp /tmp/corp-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

---
---

# PART 5: KERBEROS CONFIGURATION

---

## Step 33: Install Kerberos Packages

```bash
sudo apt update
```

```bash
sudo apt install -y krb5-user cntlm curl libgssapi-krb5-2 ntpdate
```

When prompted:
```
Default realm:        CORP.COM
KDC server:           dc1.corp.com
Admin server:         dc1.corp.com
```

---

## Step 34: Configure Kerberos

```bash
sudo nano /etc/krb5.conf
```

Replace entire contents:

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

## Step 35: Sync Time with Domain Controller

```bash
sudo ntpdate dc1.corp.com
```

```bash
sudo timedatectl set-timezone Africa/Nairobi
```

```bash
sudo timedatectl set-ntp true
```

```bash
date
```

---

## Step 36: Verify KDC is Reachable

```bash
ping -c 3 dc1.corp.com
```

```bash
nmap -p 88 dc1.corp.com
```

Port 88 must show open.

---

## Step 37: Get Kerberos Ticket

```bash
kinit youruser@CORP.COM
```

Verify:

```bash
klist
```

---

## Step 38: Test Kerberos Against Proxy

```bash
curl -v --proxy-negotiate -u : -x http://proxy2.corp.com:8080 https://google.com
```

`HTTP/1.1 200 OK` = Kerberos working directly.

---
---

# PART 6: CNTLM BRIDGE CONFIGURATION

---

## Step 39: Configure Cntlm

```bash
sudo nano /etc/cntlm.conf
```

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

## Step 40: Start Cntlm

```bash
sudo systemctl enable cntlm
```

```bash
sudo systemctl start cntlm
```

```bash
sudo systemctl status cntlm
```

Test:

```bash
curl -x http://localhost:3128 -I https://google.com
```

---
---

# PART 7: CONFIGURE ALL TOOLS

---

## Step 41: System-Wide Proxy

```bash
sudo nano /etc/environment
```

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

```bash
source /etc/environment
```

```bash
printenv | grep -i proxy
```

---

## Step 42: apt

```bash
sudo nano /etc/apt/apt.conf.d/99proxy
```

```
Acquire::http::Proxy "http://localhost:3128";
Acquire::https::Proxy "http://localhost:3128";
Acquire::ftp::Proxy "http://localhost:3128";
```

```bash
sudo apt update
```

---

## Step 43: curl

```bash
nano ~/.curlrc
```

```
proxy = "http://localhost:3128"
cacert = /etc/ssl/certs/ca-certificates.crt
```

```bash
curl -I https://google.com
```

---

## Step 44: wget

```bash
nano ~/.wgetrc
```

```
http_proxy = http://localhost:3128
https_proxy = http://localhost:3128
use_proxy = on
ca_certificate = /etc/ssl/certs/ca-certificates.crt
```

```bash
wget -q --spider https://google.com && echo "wget OK"
```

---

## Step 45: git

```bash
git config --global http.proxy http://localhost:3128
```

```bash
git config --global https.proxy http://localhost:3128
```

```bash
git config --global http.sslCAInfo /etc/ssl/certs/ca-certificates.crt
```

```bash
git ls-remote https://github.com/git/git HEAD
```

---

## Step 46: pip

```bash
mkdir -p ~/.config/pip
```

```bash
nano ~/.config/pip/pip.conf
```

```
[global]
proxy = http://localhost:3128
cert = /etc/ssl/certs/ca-certificates.crt
trusted-host = pypi.org
               pypi.python.org
               files.pythonhosted.org
```

```bash
pip install requests --dry-run
```

---

## Step 47: snap

```bash
sudo snap set system proxy.http="http://localhost:3128"
```

```bash
sudo snap set system proxy.https="http://localhost:3128"
```

```bash
sudo snap get system proxy
```

---

## Step 48: Root User

```bash
sudo nano /root/.bashrc
```

```bash
export http_proxy="http://localhost:3128"
export https_proxy="http://localhost:3128"
export HTTP_PROXY="http://localhost:3128"
export HTTPS_PROXY="http://localhost:3128"
export no_proxy="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
export NO_PROXY="localhost,127.0.0.1,10.0.0.0/8,192.168.0.0/16"
```

```bash
sudo visudo
```

Find `Defaults env_reset` and add below:

```
Defaults env_keep += "http_proxy https_proxy ftp_proxy HTTP_PROXY HTTPS_PROXY FTP_PROXY no_proxy NO_PROXY"
```

---
---

# PART 8: TICKET AUTO-RENEWAL

---

## Step 49: Create Renewal Cron Job

```bash
sudo nano /etc/cron.hourly/krb5-renew
```

```bash
#!/bin/bash
export KRB5CCNAME=/tmp/krb5cc_$(id -u)
kinit -R 2>/dev/null
if [ $? -ne 0 ]; then
    echo "Ticket renewal failed at $(date)" >> /var/log/krb5-renew.log
fi
```

```bash
sudo chmod +x /etc/cron.hourly/krb5-renew
```

Manual renewal anytime:

```bash
kinit -R
```

```bash
klist
```

---
---

# PART 9: FINAL VERIFICATION

---

## Step 50: Full Diagnostic

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
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt /usr/local/share/ca-certificates/corp-ca.crt
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

## Step 51: Save Diagnostic Script

```bash
sudo nano /usr/local/bin/netcheck.sh
```

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
echo "[8] KERBEROS TICKET"
klist 2>/dev/null || echo "No ticket - run: kinit user@CORP.COM"

echo ""
echo "[9] CERTIFICATE VERIFICATION"
openssl verify -CAfile /etc/ssl/certs/ca-certificates.crt /usr/local/share/ca-certificates/corp-ca.crt

echo ""
echo "[10] CNTLM STATUS"
sudo systemctl status cntlm | grep -E "Active|running|failed"

echo ""
echo "[11] CURL TEST"
curl -sx http://localhost:3128 -o /dev/null -w "HTTP Status: %{http_code}\n" https://google.com

echo ""
echo "[12] APT PROXY CONFIG"
cat /etc/apt/apt.conf.d/99proxy

echo ""
echo "[13] CERTIFICATE STORE"
ls /usr/local/share/ca-certificates/ | grep corp

echo ""
echo "================================================"
echo "              END OF REPORT"
echo "================================================"
```

```bash
sudo chmod +x /usr/local/bin/netcheck.sh
```

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
| 4 | Domain + DC from echo %USERDNSDOMAIN% | ☐ |
| 5 | DNS servers from ipconfig /all | ☐ |
| 6 | Corp CA cert exported as .crt AND .der | ☐ |
| 7 | VirtualBox adapter set to NAT | ☐ |
| 8 | Shared folder configured and persistent in fstab | ☐ |
| 9 | Kali has IP 10.0.2.x | ☐ |
| 10 | Gateway ping 10.0.2.2 works | ☐ |
| 11 | DNS locked in /etc/resolv.conf | ☐ |
| 12 | DC reachable from Kali | ☐ |
| 13 | Corp CA cert copied to /usr/local/share/ca-certificates/ | ☐ |
| 14 | Corp CA cert copied to /etc/ssl/certs/ | ☐ |
| 15 | update-ca-certificates ran successfully | ☐ |
| 16 | openssl verify returns OK | ☐ |
| 17 | Cert installed for curl | ☐ |
| 18 | Cert installed for wget | ☐ |
| 19 | Cert installed for git | ☐ |
| 20 | Cert installed for pip | ☐ |
| 21 | Cert installed for Java (keytool) | ☐ |
| 22 | Cert installed for Firefox | ☐ |
| 23 | Cert installed for Chromium | ☐ |
| 24 | krb5-user + cntlm installed | ☐ |
| 25 | /etc/krb5.conf configured | ☐ |
| 26 | Time synced with DC | ☐ |
| 27 | kinit succeeds and klist shows ticket | ☐ |
| 28 | Cntlm running on localhost:3128 | ☐ |
| 29 | /etc/environment points to localhost:3128 | ☐ |
| 30 | apt, curl, wget, git, pip all configured | ☐ |
| 31 | Root user proxy set | ☐ |
| 32 | sudo preserves proxy env vars | ☐ |
| 33 | Kerberos ticket auto-renewal cron set | ☐ |
| 34 | netcheck.sh passes all checks | ☐ |

---

> The two things missing from the GitHub version were: **per-tool cert pinning** (Steps 25–31) and **persistent shared folder mount via fstab** (Step 21). Without those, tools like git, pip, and Java-based tools silently fail on SSL even after `update-ca-certificates` runs.
