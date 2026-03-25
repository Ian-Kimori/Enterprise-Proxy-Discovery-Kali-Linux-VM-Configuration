# Enterprise Proxy Discovery & Kali Linux VM Configuration
### Complete Sysadmin Reference Guide

---

# PART 1: PROXY DISCOVERY ON WINDOWS

---

## 1.1 Dump All Proxy Settings at Once

```cmd
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings"
```

Full output looks like this:
```
ProxyEnable        REG_DWORD    0x1
ProxyServer        REG_SZ       proxy2.corp.com:8080
ProxyOverride      REG_SZ       localhost;127.*;10.*;192.168.*;<local>
AutoConfigURL      REG_SZ       http://wpad.corp.com/proxy.pac
```

---

## 1.2 Check System-Level Proxy (WinHTTP — separate from user proxy)

```cmd
netsh winhttp show proxy
```

Output:
```
Current WinHTTP proxy settings:
    Proxy Server(s) :  proxy2.corp.com:8080
    Bypass List     :  <local>
```

---

## 1.3 Resolve Proxy Hostname to IP

```cmd
nslookup proxy2.corp.com
```

Output:
```
Name:    proxy2.corp.com
Address: 10.10.1.5
```

Write down both the **hostname** and **IP** — you may need the IP if DNS doesn't work inside the VM.

---

## 1.4 Get Full Network Details (DNS, Gateway, IP)

```cmd
ipconfig /all
```

Note down:
```
IPv4 Address       : 10.10.0.25
Subnet Mask        : 255.255.255.0
Default Gateway    : 10.10.0.1
DNS Servers        : 10.10.1.1
                     10.10.1.2
```

---

## 1.5 Verify Proxy Port is Reachable

```cmd
telnet proxy2.corp.com 8080
```

If telnet is not enabled:

```cmd
Test-NetConnection -ComputerName proxy2.corp.com -Port 8080
```

Successful output:
```
TcpTestSucceeded : True
```

---

## 1.6 Read the PAC File (if AutoConfigURL exists)

Open in browser:
```
http://wpad.corp.com/proxy.pac
```

Or fetch via CMD:
```cmd
curl http://wpad.corp.com/proxy.pac
```

Look for lines like:
```javascript
return "PROXY proxy2.corp.com:8080";
return "PROXY 10.10.1.5:3128";
DIRECT  // means no proxy for that destination
```

PAC file tells you:
- All proxy addresses used
- Which destinations bypass the proxy
- Whether different proxies are used for different sites

---

## 1.7 Check if Proxy Requires Authentication

```cmd
curl -v -x http://proxy2.corp.com:8080 http://example.com
```

If you see:
```
HTTP/1.1 407 Proxy Authentication Required
Proxy-Authenticate: NTLM
```
→ Proxy uses **Windows domain authentication (NTLM)** — needs special handling in Kali (covered in Part 3)

If you see:
```
HTTP/1.1 200 OK
```
→ Proxy is **open** — no auth needed

---

## 1.8 Export Corporate CA Certificate

```
Win+R → certmgr.msc
```
```
Trusted Root Certification Authorities → Certificates
```
```
Find your company certificate (issued by corp CA, not public CA)
Right-click → All Tasks → Export
```
```
Format: Base-64 encoded X.509 (.CER)
Filename: corp-ca.crt
Save to Desktop
```

---

## 1.9 Master Reference Table — Fill This In Before Touching Kali

| Setting | Value |
|---|---|
| Proxy hostname | `proxy2.corp.com` |
| Proxy IP | `10.10.1.5` |
| Proxy port | `8080` |
| Proxy auth required | Yes / No |
| Auth type | NTLM / Basic / None |
| DNS server 1 | `10.10.1.1` |
| DNS server 2 | `10.10.1.2` |
| Default gateway | `10.10.0.1` |
| PAC file URL | `http://wpad.corp.com/proxy.pac` |
| Bypass list | `localhost;127.*;10.*` |
| Corp CA cert | `corp-ca.crt` |

---
---

# PART 2: VIRTUALBOX CONFIGURATION

---

## 2.1 Set Network Adapter to NAT

```
VirtualBox → Select Kali VM → Settings → Network → Adapter 1
```
```
Attached to: NAT
```
```
Click Advanced → Promiscuous Mode: Allow All
```
```
Click OK
```

---

## 2.2 Configure Shared Folder (to transfer corp-ca.crt)

```
VirtualBox → Settings → Shared Folders → Click + icon
```
```
Folder Path: C:\Users\YourUser\Desktop
Folder Name: shared
Auto-mount: Yes
Make Permanent: Yes
```
```
Click OK → OK
```

---

## 2.3 Boot the VM

```
Click Start → Wait for Kali to fully boot
```

---
---

# PART 3: KALI LINUX CONFIGURATION

---

## 3.1 Verify Network Interface

```bash
ip a
```

```bash
ip route show
```

Expected output:
```
default via 10.0.2.2 dev eth0
10.0.2.0/24 dev eth0
```

If no IP address assigned:

```bash
sudo dhclient eth0
```

---

## 3.2 Restart Network Services

```bash
sudo systemctl restart NetworkManager
```

```bash
sudo systemctl restart networking
```

---

## 3.3 Test Gateway Reachability

```bash
ping -c 3 10.0.2.2
```

This confirms VirtualBox NAT gateway is reachable before touching proxy.

---

## 3.4 Configure DNS

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

Lock file from being overwritten by NetworkManager:

```bash
sudo chattr +i /etc/resolv.conf
```

Verify it is locked:

```bash
lsattr /etc/resolv.conf
```

Output should show:
```
----i---------e--- /etc/resolv.conf
```

Test DNS:

```bash
nslookup google.com
```

```bash
dig google.com
```

---

## 3.5 Import Corporate CA Certificate

Mount shared folder:

```bash
sudo mkdir -p /mnt/shared
```

```bash
sudo mount -t vboxsf shared /mnt/shared
```

Copy and install the cert:

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

## 3.6 Set System-Wide Proxy (All Users, All Tools)

```bash
sudo nano /etc/environment
```

Add:
```
http_proxy="http://proxy2.corp.com:8080"
https_proxy="http://proxy2.corp.com:8080"
ftp_proxy="http://proxy2.corp.com:8080"
HTTP_PROXY="http://proxy2.corp.com:8080"
HTTPS_PROXY="http://proxy2.corp.com:8080"
FTP_PROXY="http://proxy2.corp.com:8080"
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

## 3.7 Set Proxy for apt

```bash
sudo nano /etc/apt/apt.conf.d/99proxy
```

Add:
```
Acquire::http::Proxy "http://proxy2.corp.com:8080";
Acquire::https::Proxy "http://proxy2.corp.com:8080";
Acquire::ftp::Proxy "http://proxy2.corp.com:8080";
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Test:

```bash
sudo apt update
```

---

## 3.8 Set Proxy for curl

```bash
nano ~/.curlrc
```

Add:
```
proxy = "http://proxy2.corp.com:8080"
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Test:

```bash
curl -sv https://google.com 2>&1 | head -20
```

---

## 3.9 Set Proxy for wget

```bash
nano ~/.wgetrc
```

Add:
```
http_proxy = http://proxy2.corp.com:8080
https_proxy = http://proxy2.corp.com:8080
ftp_proxy = http://proxy2.corp.com:8080
use_proxy = on
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

Test:

```bash
wget -q --spider https://google.com && echo "wget OK"
```

---

## 3.10 Set Proxy for git

```bash
git config --global http.proxy http://proxy2.corp.com:8080
```

```bash
git config --global https.proxy http://proxy2.corp.com:8080
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

## 3.11 Set Proxy for pip

```bash
mkdir -p ~/.config/pip
```

```bash
nano ~/.config/pip/pip.conf
```

Add:
```
[global]
proxy = http://proxy2.corp.com:8080
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

## 3.12 Set Proxy for snap

```bash
sudo snap set system proxy.http="http://proxy2.corp.com:8080"
```

```bash
sudo snap set system proxy.https="http://proxy2.corp.com:8080"
```

Verify:

```bash
sudo snap get system proxy
```

---

## 3.13 Set Proxy for Root User

```bash
sudo nano /root/.bashrc
```

Add at the bottom:
```bash
export http_proxy="http://proxy2.corp.com:8080"
export https_proxy="http://proxy2.corp.com:8080"
export HTTP_PROXY="http://proxy2.corp.com:8080"
export HTTPS_PROXY="http://proxy2.corp.com:8080"
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

## 3.14 Handle NTLM Proxy Authentication (If Proxy Returned 407)

If proxy requires Windows domain auth, install **Cntlm** as a local auth bridge:

```bash
sudo apt install cntlm
```

```bash
sudo nano /etc/cntlm.conf
```

Edit these fields:
```
Username        yourdomainuser
Domain          CORP
Password        yourpassword
Proxy           proxy2.corp.com:8080
Listen          3128
```

Generate password hash (more secure than plaintext):

```bash
cntlm -H -u yourdomainuser -d CORP
```

Replace `Password` line in config with the generated `PassNTLMv2` hash.

Start and enable Cntlm:

```bash
sudo systemctl enable cntlm
```

```bash
sudo systemctl start cntlm
```

Now update **all proxy settings** in previous steps to point to:
```
http://localhost:3128
```

Instead of the corporate proxy directly.

---

# PART 4: FINAL VERIFICATION

---

## 4.1 Run Full Diagnostic

```bash
echo "=== Network Interface ===" && ip a
```

```bash
echo "=== Routing Table ===" && ip route show
```

```bash
echo "=== DNS ===" && cat /etc/resolv.conf
```

```bash
echo "=== Proxy Env Vars ===" && printenv | grep -i proxy
```

```bash
echo "=== Gateway Ping ===" && ping -c 2 10.0.2.2
```

```bash
echo "=== DNS Test ===" && nslookup google.com
```

```bash
echo "=== curl Test ===" && curl -I https://google.com
```

```bash
echo "=== wget Test ===" && wget -q --spider https://google.com && echo "wget OK"
```

```bash
echo "=== apt Test ===" && sudo apt update
```

```bash
echo "=== git Test ===" && git ls-remote https://github.com/git/git HEAD
```

```bash
echo "=== pip Test ===" && pip install requests --dry-run
```

---

## 4.2 One-Shot Full Diagnostic Script

```bash
sudo nano /usr/local/bin/netcheck.sh
```

Paste:

```bash
#!/bin/bash
echo "================================================"
echo "       NETWORK DIAGNOSTIC REPORT"
echo "================================================"
echo ""
echo "[1] INTERFACE & IP"
ip a | grep -E "inet|state"
echo ""
echo "[2] DEFAULT ROUTE"
ip route show default
echo ""
echo "[3] DNS SERVERS"
cat /etc/resolv.conf | grep nameserver
echo ""
echo "[4] PROXY ENVIRONMENT"
printenv | grep -i proxy
echo ""
echo "[5] GATEWAY REACHABILITY"
ping -c 2 10.0.2.2 | tail -2
echo ""
echo "[6] DNS RESOLUTION"
nslookup google.com | tail -4
echo ""
echo "[7] HTTP CONNECTIVITY"
curl -sx http://proxy2.corp.com:8080 -o /dev/null -w "HTTP Status: %{http_code}\n" https://google.com
echo ""
echo "[8] APT PROXY CONFIG"
cat /etc/apt/apt.conf.d/99proxy
echo ""
echo "================================================"
echo "       END OF REPORT"
echo "================================================"
```

Save: `Ctrl+O` → `Enter` → `Ctrl+X`

```bash
sudo chmod +x /usr/local/bin/netcheck.sh
```

```bash
sudo netcheck.sh
```

Run this anytime to get a full status report instantly.

---

# MASTER CHECKLIST

| # | Task | Status |
|---|---|---|
| 1 | Proxy hostname + port discovered via reg query | ☐ |
| 2 | Proxy IP resolved via nslookup | ☐ |
| 3 | DNS servers noted from ipconfig /all | ☐ |
| 4 | PAC file read and all proxies identified | ☐ |
| 5 | Proxy auth type confirmed (407 check) | ☐ |
| 6 | Corporate CA cert exported from certmgr.msc | ☐ |
| 7 | VirtualBox adapter set to NAT | ☐ |
| 8 | Shared folder configured for cert transfer | ☐ |
| 9 | Kali has IP 10.0.2.x | ☐ |
| 10 | DNS locked in /etc/resolv.conf | ☐ |
| 11 | Corporate CA cert imported | ☐ |
| 12 | /etc/environment proxy set | ☐ |
| 13 | apt proxy configured | ☐ |
| 14 | curl, wget, git, pip proxy configured | ☐ |
| 15 | Root user proxy set | ☐ |
| 16 | sudo preserves proxy env vars | ☐ |
| 17 | Cntlm configured (if NTLM auth required) | ☐ |
| 18 | netcheck.sh passes all checks | ☐ |

---

> **Every instance of `proxy2.corp.com:8080`** must be replaced with the exact values from your Part 1 discovery. The proxy hostname/IP and port you find in Step 1.3 drives every configuration that follows.
