# OpenSSH Server on Windows — SIT Environment Guide

คู่มือติดตั้งและตั้งค่า OpenSSH Server บน Windows สำหรับใช้งานเป็น SIT Deploy Server
ให้ใกล้เคียงกับ Linux Server ในบริษัท พร้อมรองรับการเข้าถึงจากวง LAN และผ่าน VPN จากภายนอก

---

## สารบัญ

1. [ติดตั้ง OpenSSH Server](#1-ติดตั้ง-openssh-server)
2. [ตั้งค่า sshd_config](#2-ตั้งค่า-sshd_config)
3. [ตั้งค่า Firewall](#3-ตั้งค่า-firewall)
4. [Key-based Authentication (ไม่ใช้ Password)](#4-key-based-authentication)
5. [ตั้งค่า Default Shell เป็น Bash (Git Bash / WSL)](#5-ตั้งค่า-default-shell)
6. [สร้าง User สำหรับ Deploy (deploy user)](#6-สร้าง-deploy-user)
7. [ทดสอบเชื่อมต่อใน LAN เดียวกัน](#7-ทดสอบเชื่อมต่อใน-lan)
8. [การเข้าถึงจากภายนอกผ่าน VPN](#8-vpn-สำหรับเข้าจากภายนอก)
9. [เพิ่มความปลอดภัยเพิ่มเติม (Hardening)](#9-security-hardening)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. ติดตั้ง OpenSSH Server

### วิธีที่ 1 — ผ่าน Settings (GUI)

```
Settings → Apps → Optional Features → Add a Feature → ค้นหา "OpenSSH Server" → Install
```

### วิธีที่ 2 — ผ่าน PowerShell (แนะนำ)

เปิด PowerShell ในฐานะ **Administrator** แล้วรันคำสั่งต่อไปนี้:

```powershell
# ตรวจสอบว่ามีหรือยัง
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'

# ติดตั้ง OpenSSH Server
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0

# เริ่ม Service และตั้งให้ auto-start
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic

# ตรวจสอบสถานะ
Get-Service sshd
```

---

## 2. ตั้งค่า sshd_config

ไฟล์ config อยู่ที่: `C:\ProgramData\ssh\sshd_config`

เปิดแก้ไขด้วย PowerShell Administrator:

```powershell
notepad C:\ProgramData\ssh\sshd_config
```

### การตั้งค่าที่แนะนำ (เทียบกับ Linux SIT Server)

```sshd_config
# =============================================
# Port & Network
# =============================================
Port 22
#Port 2222        # สำรองไว้ถ้าต้องการเปลี่ยน port

AddressFamily inet
ListenAddress 0.0.0.0

# =============================================
# Authentication
# =============================================
LoginGraceTime 30
PermitRootLogin no
StrictModes yes
MaxAuthTries 5
MaxSessions 10

# เปิดใช้ Key Authentication
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# ปิด Password login (เปิดไว้ก่อนถ้ายังไม่ได้ setup key)
PasswordAuthentication yes
PermitEmptyPasswords no

# =============================================
# Session & Forwarding
# =============================================
X11Forwarding no
PrintMotd yes
TCPKeepAlive yes
ClientAliveInterval 60
ClientAliveCountMax 3

# =============================================
# SFTP Subsystem
# =============================================
Subsystem sftp sftp-server.exe

# =============================================
# Allow Users (จำกัดเฉพาะ user ที่กำหนด)
# =============================================
AllowUsers deploy administrator

# =============================================
# Banner — แสดงข้อความตอน login (optional)
# =============================================
# Banner C:\ProgramData\ssh\banner.txt
```

หลังแก้ไข ให้ restart service:

```powershell
Restart-Service sshd
```

---

## 3. ตั้งค่า Firewall

Windows Firewall จะถูกเพิ่ม rule อัตโนมัติตอนติดตั้ง แต่ให้ตรวจสอบและล็อค Source IP ให้ตรงกับ subnet ของบริษัท:

```powershell
# ตรวจสอบ rule ที่มีอยู่
Get-NetFirewallRule -Name *ssh*

# ลบ rule เดิมที่เปิดทุก IP
Remove-NetFirewallRule -Name "OpenSSH-Server-In-TCP"

# สร้าง rule ใหม่ — อนุญาตเฉพาะ LAN subnet และ VPN subnet
New-NetFirewallRule `
    -Name "OpenSSH-Server-In-TCP" `
    -DisplayName "OpenSSH Server (inbound)" `
    -Enabled True `
    -Direction Inbound `
    -Protocol TCP `
    -Action Allow `
    -LocalPort 22 `
    -RemoteAddress 192.168.1.0/24, 10.8.0.0/24
# แก้ 192.168.1.0/24 เป็น subnet LAN จริง
# แก้ 10.8.0.0/24 เป็น subnet VPN จริง
```

ตรวจสอบ IP ของเครื่องตัวเอง:

```powershell
ipconfig
# ดู IPv4 Address ของ adapter ที่ต่อ LAN
```

---

## 4. Key-based Authentication

### สร้าง SSH Key pair บนเครื่อง Client (เครื่องที่จะ SSH เข้ามา)

```bash
# รันบนเครื่อง client (Linux/Mac/Windows Git Bash)
ssh-keygen -t ed25519 -C "sit-deploy-key" -f ~/.ssh/sit_deploy_key
```

### คัดลอก Public Key ไปวางบนเครื่อง Server (Windows)

```powershell
# บน Windows Server — สร้าง folder .ssh ให้ user deploy
New-Item -ItemType Directory -Path "C:\Users\deploy\.ssh" -Force

# วาง public key (เนื้อหาของ sit_deploy_key.pub) ลงในไฟล์นี้
notepad C:\Users\deploy\.ssh\authorized_keys
```

### แก้ไข Permission ของ authorized_keys (สำคัญมาก)

Windows ต้องการ permission ที่เข้มงวด ไม่เช่นนั้น SSH จะ ignore key:

```powershell
# ตั้ง permission ให้ถูกต้อง
$file = "C:\Users\deploy\.ssh\authorized_keys"
$acl = Get-Acl $file

# ลบ inheritance
$acl.SetAccessRuleProtection($true, $false)

# ให้สิทธิ์เฉพาะ user deploy และ SYSTEM
$rule1 = New-Object System.Security.AccessControl.FileSystemAccessRule("deploy","FullControl","Allow")
$rule2 = New-Object System.Security.AccessControl.FileSystemAccessRule("SYSTEM","FullControl","Allow")
$acl.SetAccessRule($rule1)
$acl.SetAccessRule($rule2)
Set-Acl $file $acl
```

### ทดสอบ login ด้วย Key

```bash
# จากเครื่อง client
ssh -i ~/.ssh/sit_deploy_key deploy@192.168.1.xxx
```

---

## 5. ตั้งค่า Default Shell

ค่าเริ่มต้น SSH บน Windows จะเปิด `cmd.exe` ซึ่งต่างจาก Linux มาก แนะนำให้เปลี่ยนเป็น PowerShell หรือ Bash (WSL)

### Option A — PowerShell (แนะนำสำหรับ Windows-native)

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" `
    -Name DefaultShell `
    -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" `
    -PropertyType String `
    -Force
```

### Option B — Git Bash (bash.exe จาก Git for Windows)

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" `
    -Name DefaultShell `
    -Value "C:\Program Files\Git\bin\bash.exe" `
    -PropertyType String `
    -Force
```

### Option C — WSL Bash (เหมือน Linux มากที่สุด)

ต้องติดตั้ง WSL ก่อน:

```powershell
wsl --install -d Ubuntu
```

แล้วชี้ default shell:

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" `
    -Name DefaultShell `
    -Value "C:\Windows\System32\wsl.exe" `
    -PropertyType String `
    -Force
```

> **หมายเหตุ:** WSL shell จะให้ environment เหมือน Linux มากที่สุด เหมาะกับการ deploy script ที่เขียนสำหรับ Linux

---

## 6. สร้าง Deploy User

```powershell
# สร้าง local user สำหรับ deploy โดยเฉพาะ
$password = Read-Host -AsSecureString "Enter password for deploy user"
New-LocalUser -Name "deploy" -Password $password -FullName "SIT Deploy User" -Description "SSH deploy account"

# เพิ่มเข้า group ตามความจำเป็น (อย่าใส่ Administrators ถ้าไม่จำเป็น)
Add-LocalGroupMember -Group "Users" -Member "deploy"

# ถ้า deploy ต้องรัน script ที่ต้องการสิทธิ์เพิ่มเติม
# Add-LocalGroupMember -Group "Administrators" -Member "deploy"
```

### ตั้ง Working Directory สำหรับ Deploy

```powershell
# สร้าง folder สำหรับ deploy artifacts
New-Item -ItemType Directory -Path "C:\sit-deploy" -Force

# ให้สิทธิ์ user deploy
$acl = Get-Acl "C:\sit-deploy"
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule("deploy","Modify","ContainerInherit,ObjectInherit","None","Allow")
$acl.SetAccessRule($rule)
Set-Acl "C:\sit-deploy" $acl
```

---

## 7. ทดสอบเชื่อมต่อใน LAN

### ตรวจสอบ IP เครื่อง Server

```powershell
ipconfig | Select-String "IPv4"
# ตัวอย่าง: 192.168.1.50
```

### SSH เข้าจากเครื่อง Client ในวงเดียวกัน

```bash
# Password login
ssh deploy@192.168.1.50

# Key login
ssh -i ~/.ssh/sit_deploy_key deploy@192.168.1.50

# ระบุ port ถ้าเปลี่ยนจาก 22
ssh -p 2222 -i ~/.ssh/sit_deploy_key deploy@192.168.1.50
```

### SFTP (upload/download files)

```bash
sftp deploy@192.168.1.50
# หรือ
sftp -i ~/.ssh/sit_deploy_key deploy@192.168.1.50
```

### SCP (copy file โดยตรง)

```bash
# อัปโหลดไฟล์ไปยัง server
scp -i ~/.ssh/sit_deploy_key ./app.zip deploy@192.168.1.50:/sit-deploy/

# ดาวน์โหลดไฟล์จาก server
scp -i ~/.ssh/sit_deploy_key deploy@192.168.1.50:/sit-deploy/logs/app.log ./
```

---

## 8. VPN สำหรับเข้าจากภายนอก

เครื่อง PC/Laptop ที่รัน OpenSSH Server ต้องอยู่ใน LAN เดียวกับ VPN Server หรือเข้าถึงได้จาก VPN subnet

### สถาปัตยกรรมที่แนะนำ

```
[Developer นอกออฟฟิศ]
        │
        ▼ (VPN Client เชื่อมต่อ)
[VPN Server / Router ที่ออฟฟิศ]
        │
        ▼ (Tunnel → LAN)
[PC/Laptop รัน OpenSSH Server]
  IP: 192.168.1.50
```

### Option A — ใช้ OpenVPN (Self-hosted)

#### ติดตั้ง OpenVPN Access Server หรือ Community Edition บน Router/Server

```bash
# บน Linux VPN Gateway (ตัวอย่าง Ubuntu)
apt install openvpn easy-rsa

# สร้าง CA และ certificates
make-cadir ~/openvpn-ca && cd ~/openvpn-ca
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa gen-dh
./easyrsa gen-req server nopass
./easyrsa sign-req server server
./easyrsa gen-req client1 nopass
./easyrsa sign-req client client1
```

#### ตัวอย่าง server.conf (OpenVPN)

```conf
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
server 10.8.0.0 255.255.255.0
push "route 192.168.1.0 255.255.255.0"
keepalive 10 120
persist-key
persist-tun
verb 3
```

#### กำหนด Firewall บน VPN Server ให้ forward traffic ไป SSH Server

```bash
# บน Linux VPN Gateway
iptables -A FORWARD -i tun0 -o eth0 -d 192.168.1.50 -p tcp --dport 22 -j ACCEPT
iptables -A FORWARD -i eth0 -o tun0 -s 192.168.1.50 -p tcp --sport 22 -j ACCEPT
```

### Option B — WireGuard (เร็วกว่า OpenVPN แนะนำ)

#### ติดตั้ง WireGuard บน Windows Server (เครื่อง SSH)

```powershell
# ดาวน์โหลด installer จาก wireguard.com แล้วรัน
# หรือใช้ winget
winget install WireGuard.WireGuard
```

#### ตัวอย่าง wg0.conf บน Windows (Peer/Client mode)

```ini
[Interface]
PrivateKey = <private_key_ของเครื่องนี้>
Address = 10.8.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = <public_key_ของ_vpn_server>
AllowedIPs = 10.8.0.0/24
Endpoint = <vpn_server_public_ip>:51820
PersistentKeepalive = 25
```

#### Developer ที่เชื่อมต่อผ่าน WireGuard จะใช้ IP ที่ได้รับ (เช่น 10.8.0.x) SSH เข้าได้เลย

```bash
ssh -i ~/.ssh/sit_deploy_key deploy@10.8.0.2
```

### Option C — ใช้ VPN ที่มีอยู่แล้วในบริษัท (Cisco AnyConnect, FortiClient, etc.)

1. เชื่อมต่อ VPN ด้วย client ของบริษัท
2. ตรวจสอบ IP ที่ได้รับจาก VPN (มักเป็น 10.x.x.x หรือ 172.16.x.x)
3. SSH ตรงไปยัง IP ของเครื่อง Server ใน LAN ได้เลย (ถ้า routing ถูกต้อง)

```bash
ssh deploy@192.168.1.50
```

> **หมายเหตุ:** ต้องให้ Network Admin ของบริษัทเปิด route จาก VPN subnet ไปยัง IP ของเครื่อง

---

## 9. Security Hardening

### ล็อค Port 22 ด้วย IP Whitelist (ทำซ้ำจากข้อ 3 เพื่อเน้นย้ำ)

```powershell
# อนุญาตเฉพาะ LAN + VPN subnet
New-NetFirewallRule -Name "SSH-Whitelist" `
    -DisplayName "SSH Whitelist" `
    -Enabled True -Direction Inbound -Protocol TCP `
    -Action Allow -LocalPort 22 `
    -RemoteAddress "192.168.1.0/24","10.8.0.0/24"
```

### ปิด Password Authentication หลังจาก Setup Key แล้ว

แก้ใน `C:\ProgramData\ssh\sshd_config`:

```sshd_config
PasswordAuthentication no
```

### ตั้งค่า Fail2ban-equivalent บน Windows — Audit Login Failures

```powershell
# ดู login failure events
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625} -MaxEvents 20 |
    Select-Object TimeCreated, Message
```

### เปลี่ยน SSH Port (optional, ลด noise จาก bot scan)

```sshd_config
Port 2222
```

อย่าลืมอัปเดต Firewall rule ด้วย:

```powershell
New-NetFirewallRule -Name "SSH-Custom-Port" `
    -DisplayName "SSH Custom Port 2222" `
    -Enabled True -Direction Inbound -Protocol TCP `
    -Action Allow -LocalPort 2222 `
    -RemoteAddress "192.168.1.0/24","10.8.0.0/24"
```

### Banner แสดงข้อความเตือน (เหมือน Production Linux Server)

สร้างไฟล์ `C:\ProgramData\ssh\banner.txt`:

```
*******************************************************************************
  WARNING: Authorized access only. SIT Environment — Works Lab
  Unauthorized access is prohibited and will be prosecuted.
*******************************************************************************
```

เปิดใช้ใน sshd_config:

```sshd_config
Banner C:\ProgramData\ssh\banner.txt
```

---

## 10. Troubleshooting

### SSH ไม่ตอบสนอง

```powershell
# ตรวจสอบ service
Get-Service sshd

# ดู log ของ sshd
Get-WinEvent -LogName "OpenSSH/Operational" -MaxEvents 30

# หรือดู Windows Event Log
eventvwr.msc
# → Windows Logs → Application → กรองด้วย "OpenSSH"
```

### Connection Refused

```powershell
# ตรวจสอบว่า port 22 เปิดอยู่
netstat -ano | Select-String ":22"

# ตรวจสอบ Firewall rule
Get-NetFirewallRule -Name *ssh* | Select-Object DisplayName, Enabled, Direction, Action
```

### Permission denied (publickey)

```powershell
# ตรวจสอบ permission ของ authorized_keys
Get-Acl "C:\Users\deploy\.ssh\authorized_keys" | Format-List

# ดู SSH debug log บน server
# แก้ใน sshd_config ชั่วคราว:
# LogLevel DEBUG3
# แล้ว restart sshd และดูที่ Event Viewer
```

### SSH เข้าได้แต่ Command ไม่ทำงานเหมือน Linux

```powershell
# ตรวจสอบ default shell ที่ตั้งไว้
Get-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell
```

### ทดสอบ Connection จาก Client แบบ Verbose

```bash
ssh -vvv deploy@192.168.1.50
```

---

## สรุป Architecture

```
[Developer ในออฟฟิศ / LAN 192.168.1.0/24]
    └─── SSH port 22 ──────────────────────────┐
                                               ▼
[Developer นอกออฟฟิศ]                [Windows PC/Laptop]
    └─── VPN ───► [VPN Gateway]        IP: 192.168.1.50
                       └─── route ──►  sshd running
                                       User: deploy
                                       Shell: PowerShell / Git Bash / WSL
                                       Deploy dir: C:\sit-deploy\
```

---

## Checklist ก่อนใช้งานจริง

- [ ] ติดตั้ง OpenSSH Server และ service เป็น Automatic
- [ ] แก้ `sshd_config` ตามที่กำหนด
- [ ] ตั้ง Firewall อนุญาตเฉพาะ LAN + VPN subnet
- [ ] สร้าง user `deploy` และ folder `C:\sit-deploy`
- [ ] Setup SSH Key authentication และแก้ permission authorized_keys
- [ ] เปลี่ยน default shell เป็น PowerShell หรือ Git Bash
- [ ] ทดสอบ SSH จากในวง LAN
- [ ] เชื่อมต่อ VPN และทดสอบ SSH จากภายนอก
- [ ] ปิด PasswordAuthentication หลัง Key ใช้งานได้แล้ว
- [ ] เพิ่ม Banner และ logging
