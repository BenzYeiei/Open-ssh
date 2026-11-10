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
7. [ติดตั้งและตั้งค่า SSH Client + ทดสอบเชื่อมต่อใน LAN](#7-ติดตั้งและตั้งค่า-ssh-client--ทดสอบเชื่อมต่อใน-lan)
   - [7.1 ติดตั้ง SSH Client](#71-ติดตั้ง-ssh-client-บนเครื่อง-client)
   - [7.2 สร้าง SSH Key Pair](#72-สร้าง-ssh-key-pair-บนเครื่อง-client)
   - [7.3 ส่ง Public Key ให้ Server Admin](#73-ส่ง-public-key-ให้-server-admin)
   - [7.4 ตั้งค่า SSH Config File](#74-ตั้งค่า-ssh-config-file-แนะนำ--ไม่ต้องพิมพ์-option-ทุกครั้ง)
   - [7.5 ตรวจสอบ IP เครื่อง Server](#75-ตรวจสอบ-ip-เครื่อง-server)
   - [7.6 ทดสอบ SSH เข้าจาก LAN](#76-ทดสอบ-ssh-เข้าจากเครื่อง-client-ในวงเดียวกัน)
   - [7.7 SFTP](#77-sftp-uploaddownload-files)
   - [7.8 SCP](#78-scp-copy-file-แบบ-one-liner)
   - [7.9 แก้ปัญหา Permission บน Windows](#79-แก้ปัญหา-permission-ของ-private-key-บน-windows)
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

ตอนติดตั้ง OpenSSH Server Windows จะสร้าง Firewall rule ให้อัตโนมัติ แต่ rule นั้นเปิดรับ connection จาก **ทุก IP** ซึ่งไม่ปลอดภัย ควรลบทิ้งแล้วสร้างใหม่โดยล็อคให้รับเฉพาะ subnet LAN และ VPN เท่านั้น

### หา Subnet ของเครื่องตัวเองก่อน

```powershell
ipconfig
```

ดูที่ **IPv4 Address** และ **Subnet Mask** ของ adapter ที่ต่อ LAN เช่น:
- IPv4: `192.168.1.50`
- Subnet Mask: `255.255.255.0`
- Subnet: `192.168.1.0/24` (ใช้ค่านี้ในขั้นตอนถัดไป)

### ตั้งค่า Firewall Rule

```powershell
# ตรวจสอบ rule ที่มีอยู่
Get-NetFirewallRule -Name *ssh*

# ลบ rule เดิมที่เปิดรับทุก IP
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
    -RemoteAddress "192.168.1.0/24","10.8.0.0/24"
```

> แก้ `192.168.1.0/24` เป็น subnet LAN จริงของเครื่อง
> แก้ `10.8.0.0/24` เป็น subnet VPN จริงของบริษัท

คำอธิบาย option แต่ละตัว:

| Option | ความหมาย |
|--------|----------|
| `-Name` | ชื่อ internal ของ rule ใช้อ้างอิงตอนแก้ไขหรือลบ |
| `-DisplayName` | ชื่อที่แสดงใน Windows Firewall UI |
| `-Direction Inbound` | กรอง traffic ที่เข้ามาหาเครื่องนี้ |
| `-Protocol TCP` | SSH ใช้ TCP |
| `-Action Allow` | อนุญาต connection ที่ตรงเงื่อนไข |
| `-LocalPort 22` | port ที่เปิดรับบนเครื่องนี้ |
| `-RemoteAddress` | IP หรือ subnet ที่อนุญาตให้เชื่อมต่อเข้ามา — ใส่หลายค่าได้คั่นด้วย `,` |

### ตรวจสอบว่า Rule ถูกสร้างแล้ว

```powershell
Get-NetFirewallRule -Name "OpenSSH-Server-In-TCP" | Format-List DisplayName, Enabled, Action
Get-NetFirewallAddressFilter -AssociatedNetFirewallRule (Get-NetFirewallRule -Name "OpenSSH-Server-In-TCP")
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

## 7. ติดตั้งและตั้งค่า SSH Client + ทดสอบเชื่อมต่อใน LAN

### 7.1 ติดตั้ง SSH Client บนเครื่อง Client

#### Windows

OpenSSH Client มาพร้อม Windows 10/11 แล้ว ตรวจสอบด้วย:

```powershell
# ตรวจสอบว่ามีหรือยัง
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Client*'

# ถ้าไม่มี ให้ติดตั้ง
Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
```

ทดสอบว่าใช้งานได้:

```powershell
ssh -V
# ควรแสดง: OpenSSH_for_Windows_x.x ...
```

#### macOS

macOS มี SSH Client ติดมาแล้ว ตรวจสอบด้วย:

```bash
ssh -V
```

#### Linux (Ubuntu/Debian)

```bash
sudo apt update && sudo apt install openssh-client -y
ssh -V
```

---

### 7.2 สร้าง SSH Key Pair บนเครื่อง Client

**ขั้นตอนที่ 1 — สร้าง folder `.ssh` ก่อน** (PowerShell จะไม่สร้างให้อัตโนมัติ)

```powershell
New-Item -ItemType Directory -Path "$env:USERPROFILE\.ssh" -Force
```

**ขั้นตอนที่ 2 — สร้าง Key Pair**

```powershell
ssh-keygen -t ed25519 -C "yourname-sit" -f "$env:USERPROFILE\.ssh\sit_deploy_key"
```

| Option | ความหมาย |
|--------|----------|
| `-t ed25519` | ประเภทของ algorithm — Ed25519 ปลอดภัยและเร็วกว่า RSA 4096 |
| `-C "yourname-sit"` | Comment ติดท้าย public key ไว้บอกว่า key นี้เป็นของใคร/ใช้กับอะไร |
| `-f "$env:USERPROFILE\.ssh\sit_deploy_key"` | ชื่อและ path ของไฟล์ key ที่จะสร้าง (ใช้ `$env:USERPROFILE` แทน `~` เพราะ PowerShell แปล `~` ไม่ได้ใน path ที่ส่งให้ ssh-keygen) |

หลังรันคำสั่งจะถาม **passphrase** — เป็น password ที่ใช้ล็อค private key ไว้บนเครื่องตัวเอง แนะนำให้ตั้งเสมอ เพราะถ้าไฟล์หลุดออกไปคนอื่นก็ยังใช้ไม่ได้โดยไม่รู้ passphrase

ไฟล์ที่ได้:
- `C:\Users\<username>\.ssh\sit_deploy_key` — **private key** ห้ามแชร์หรือ copy ออกนอกเครื่อง
- `C:\Users\<username>\.ssh\sit_deploy_key.pub` — **public key** ส่งให้ server admin นำไปวางบน server ได้เลย

---

### 7.3 ส่ง Public Key ให้ Server Admin

```bash
# แสดงเนื้อหา public key เพื่อ copy ส่งให้ admin
cat ~/.ssh/sit_deploy_key.pub
# ตัวอย่าง output:
# ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAA... yourname-sit
```

Admin จะนำ public key ไปวางใน `C:\Users\deploy\.ssh\authorized_keys` บน server (ดูข้อ 4)

---

### 7.4 ตั้งค่า SSH Config File (แนะนำ — ไม่ต้องพิมพ์ option ทุกครั้ง)

SSH Config File คือไฟล์ที่เก็บการตั้งค่าการเชื่อมต่อของแต่ละ server ไว้ล่วงหน้า แทนที่จะต้องพิมพ์ option ยาวๆ ทุกครั้ง เช่น `ssh -i ~/.ssh/sit_deploy_key -p 22 deploy@192.168.1.50` ก็ย่อเหลือแค่ `ssh sit`

สร้างหรือแก้ไขไฟล์ที่ `C:\Users\<username>\.ssh\config` (ไม่มีนามสกุล):

```powershell
notepad "$env:USERPROFILE\.ssh\config"
```

วางเนื้อหาดังนี้:

```sshconfig
# SIT Server (ใน LAN)
Host sit
    HostName 192.168.1.50
    User deploy
    Port 22
    IdentityFile C:\Users\<username>\.ssh\sit_deploy_key
    ServerAliveInterval 60
    ServerAliveCountMax 3

# SIT Server ผ่าน VPN (จากภายนอก)
Host sit-vpn
    HostName 192.168.1.50
    User deploy
    Port 22
    IdentityFile C:\Users\<username>\.ssh\sit_deploy_key
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

คำอธิบาย option แต่ละตัว:

| Option | ความหมาย |
|--------|----------|
| `Host sit` | ชื่อ alias ที่ใช้พิมพ์ตอน SSH เช่น `ssh sit` |
| `HostName` | IP หรือ domain จริงของ server |
| `User` | username ที่ใช้ login |
| `Port` | port ของ SSH server (default 22) |
| `IdentityFile` | path ของ private key — บน Windows ต้องใช้ path เต็ม เช่น `C:\Users\<username>\.ssh\sit_deploy_key` (ใช้ `~` ไม่ได้ใน config file บน Windows) |
| `ServerAliveInterval 60` | ส่ง keepalive packet ทุก 60 วินาที ป้องกัน connection หลุดเวลา idle |
| `ServerAliveCountMax 3` | ถ้าไม่ได้รับ response 3 ครั้งติดกัน ให้ตัด connection |

หลังจากตั้งค่าแล้ว SSH เข้าได้แบบสั้น:

```powershell
# ใน LAN
ssh sit

# ผ่าน VPN
ssh sit-vpn

# ใช้กับ SCP และ SFTP ได้ด้วย
scp ./app.zip sit:/sit-deploy/
sftp sit
```

---

### 7.5 ตรวจสอบ IP เครื่อง Server

```powershell
# รันบนเครื่อง Server
ipconfig | Select-String "IPv4"
# ตัวอย่าง: 192.168.1.50
```

---

### 7.6 ทดสอบ SSH เข้าจากเครื่อง Client ในวงเดียวกัน

```bash
# ใช้ SSH Config alias (หลังตั้งค่าข้อ 7.4)
ssh sit

# หรือพิมพ์เต็ม
ssh deploy@192.168.1.50

# Key login ระบุ identity file เอง
ssh -i ~/.ssh/sit_deploy_key deploy@192.168.1.50

# ระบุ port ถ้าเปลี่ยนจาก 22
ssh -p 2222 -i ~/.ssh/sit_deploy_key deploy@192.168.1.50
```

---

### 7.7 SFTP (upload/download files)

```bash
# ใช้ alias จาก SSH Config
sftp sit

# หรือพิมพ์เต็ม
sftp -i ~/.ssh/sit_deploy_key deploy@192.168.1.50

# คำสั่งที่ใช้บ่อยใน SFTP shell
# put localfile.zip /sit-deploy/     # อัปโหลด
# get /sit-deploy/logs/app.log ./    # ดาวน์โหลด
# ls /sit-deploy/                    # ดู directory บน server
# lls                                # ดู directory บน local
# exit
```

---

### 7.8 SCP (copy file แบบ one-liner)

```bash
# อัปโหลดไฟล์ไปยัง server
scp -i ~/.ssh/sit_deploy_key ./app.zip deploy@192.168.1.50:/sit-deploy/

# อัปโหลดทั้ง folder
scp -r -i ~/.ssh/sit_deploy_key ./dist/ deploy@192.168.1.50:/sit-deploy/

# ดาวน์โหลดไฟล์จาก server
scp -i ~/.ssh/sit_deploy_key deploy@192.168.1.50:/sit-deploy/logs/app.log ./

# ใช้ alias จาก SSH Config (สั้นกว่า)
scp ./app.zip sit:/sit-deploy/
scp sit:/sit-deploy/logs/app.log ./
```

---

### 7.9 แก้ปัญหา Permission ของ Private Key บน Windows

Windows บางครั้งให้ permission ของ private key กว้างเกินไป ทำให้ SSH ปฏิเสธการใช้งาน:

```powershell
# แก้ permission ของ private key ให้เป็นเจ้าของคนเดียว
$keyFile = "$env:USERPROFILE\.ssh\sit_deploy_key"
$acl = Get-Acl $keyFile
$acl.SetAccessRuleProtection($true, $false)
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    $env:USERNAME, "FullControl", "Allow"
)
$acl.SetAccessRule($rule)
Set-Acl $keyFile $acl
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
