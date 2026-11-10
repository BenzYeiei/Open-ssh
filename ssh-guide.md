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
8. [เพิ่ม Client User ใหม่](#8-เพิ่ม-client-user-ใหม่)
9. [การเข้าถึงจากภายนอกผ่าน VPN](#9-vpn-สำหรับเข้าจากภายนอก)
10. [เพิ่มความปลอดภัยเพิ่มเติม (Hardening)](#10-security-hardening)
11. [Troubleshooting](#11-troubleshooting)

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
AllowUsers deverloper administrator

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

**ขั้นตอนที่ 1 — หาชื่อ rule เดิม**

```powershell
Get-NetFirewallRule | Where-Object { $_.DisplayName -like "*ssh*" -or $_.DisplayName -like "*OpenSSH*" } | Select-Object Name, DisplayName
```

ชื่อ rule ที่ Windows สร้างให้อัตโนมัติอาจต่างกันในแต่ละเครื่อง เช่น:
- `OpenSSH-Server-In-TCP`
- `{AE636657-5B24-4CBD-9071-80B3BBF383B6}` (GUID)

**ขั้นตอนที่ 2 — ลบ rule เดิม** (ใช้ชื่อที่ได้จากขั้นตอนที่ 1)

```powershell
Remove-NetFirewallRule -Name "ชื่อที่ได้จากขั้นตอนที่ 1"

# ตัวอย่าง ถ้าได้ GUID
Remove-NetFirewallRule -Name "{AE636657-5B24-4CBD-9071-80B3BBF383B6}"
```

> ถ้าค้นหาแล้วไม่มี rule เลย ข้ามขั้นตอนนี้ไปได้เลย

**ขั้นตอนที่ 3 — สร้าง rule ใหม่ล็อค subnet**

```powershell
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

Key-based Authentication คือการยืนยันตัวตนด้วย SSH Key แทนการพิมพ์ password ทุกครั้ง โดยใช้หลักการ **Public/Private Key Pair**:

- **Private Key** — เก็บไว้บนเครื่อง client ห้ามแชร์
- **Public Key** — นำไปวางบน server ใน `authorized_keys` เปรียบเหมือนกุญแจล็อค

เวลา SSH เข้า server จะนำ private key มาพิสูจน์กับ public key ที่วางไว้ ถ้าตรงกันก็เข้าได้โดยไม่ต้องใช้ password

> **ข้อ 4 นี้ทำบนเครื่อง Server** — admin เป็นคนดำเนินการ
> ส่วนการสร้าง key บนเครื่อง client ดูได้ที่ข้อ 7.2

---

### ขั้นตอนที่ 1 — รับ Public Key จาก Client

ให้ client ส่งเนื้อหาของไฟล์ `sit_deploy_key.pub` มาให้ ตัวอย่างเนื้อหาที่ได้รับ:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx yourname-sit
```

---

### ขั้นตอนที่ 2 — สร้าง folder .ssh และไฟล์ authorized_keys บน Server

```powershell
# สร้าง folder .ssh ให้ user deploy
New-Item -ItemType Directory -Path "C:\Users\deverloper\.ssh" -Force

# สร้างไฟล์ authorized_keys แล้ววาง public key ที่ได้รับจาก client ลงไป
notepad C:\Users\deverloper\.ssh\authorized_keys
```

วาง public key ทั้งบรรทัดลงในไฟล์ แล้ว Save ถ้ามีหลาย client ให้วางคนละบรรทัด:

```
ssh-ed25519 AAAAC3NzaC1lZ... user1-sit
ssh-ed25519 AAAAC3NzaC1lZ... user2-sit
```

---

### ขั้นตอนที่ 3 — แก้ Permission ของ authorized_keys (สำคัญมาก)

บน Windows ถ้า permission ของไฟล์กว้างเกินไป SSH จะ **ignore** ไฟล์นี้ทันทีโดยไม่แจ้ง error ชัดเจน ต้องล็อคให้เฉพาะ user deverloper และ SYSTEM เท่านั้นที่เข้าถึงได้:

```powershell
$file = "C:\Users\deverloper\.ssh\authorized_keys"
$acl = Get-Acl $file

# ตัด permission ที่ inherit มาจาก folder แม่ออกทั้งหมด
$acl.SetAccessRuleProtection($true, $false)

# ให้สิทธิ์เฉพาะ user deverloper และ SYSTEM
$rule1 = New-Object System.Security.AccessControl.FileSystemAccessRule("deverloper","FullControl","Allow")
$rule2 = New-Object System.Security.AccessControl.FileSystemAccessRule("SYSTEM","FullControl","Allow")
$acl.SetAccessRule($rule1)
$acl.SetAccessRule($rule2)
Set-Acl $file $acl
```

ตรวจสอบว่า permission ถูกต้อง:

```powershell
Get-Acl "C:\Users\deverloper\.ssh\authorized_keys" | Format-List
```

---

### ขั้นตอนที่ 4 — ทดสอบ login ด้วย Key

```powershell
# รันจากเครื่อง client
ssh -i "$env:USERPROFILE\.ssh\sit_deploy_key" deverloper@192.168.1.50
```

ถ้าเข้าได้โดยไม่ถามรหัสผ่าน (หรือถามแค่ passphrase ของ key) แสดงว่าตั้งค่าถูกต้องแล้ว

---

## 5. ตั้งค่า Default Shell

ค่าเริ่มต้น SSH บน Windows จะเปิด `cmd.exe` ต้องเปลี่ยนเป็น PowerShell รันบน **Server** ใน PowerShell Administrator:

```powershell
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" `
    -Name DefaultShell `
    -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" `
    -PropertyType String `
    -Force
```

ตรวจสอบว่าตั้งค่าถูกต้อง:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell
```

### Oh My Posh

Server ใช้ oh-my-posh ผ่าน `microsoft.powershell_profile.ps1` ซึ่งจะโหลดทุกครั้งที่เปิด PowerShell รวมถึงตอน SSH เข้ามาด้วย

ถ้า terminal ของ client ไม่มี Nerd Font จะแสดง icon เป็นกล่องสี่เหลี่ยม แก้ได้สองวิธี:

**วิธีที่ 1 — ติดตั้ง Nerd Font บนเครื่อง client** เช่น `CaskaydiaCove NF` จาก [nerdfonts.com](https://www.nerdfonts.com) แล้วตั้งเป็น font ใน Windows Terminal

**วิธีที่ 2 — ปิด oh-my-posh เฉพาะ SSH session** แก้ไฟล์ profile บน server:

```powershell
notepad $PROFILE
```

เพิ่มเงื่อนไขครอบ oh-my-posh:

```powershell
if (-not $env:SSH_CONNECTION) {
    oh-my-posh init pwsh | Invoke-Expression
}
```

`$env:SSH_CONNECTION` จะมีค่าเฉพาะตอนเข้าผ่าน SSH ถ้านั่งใช้หน้าเครื่องตรงๆ จะเป็นค่าว่าง ทำให้ oh-my-posh ทำงานเฉพาะหน้าเครื่องเท่านั้น

---

## 6. สร้าง Deploy User

แนะนำให้สร้าง user แยกสำหรับ SSH โดยเฉพาะ ไม่ใช้ Administrator account เพื่อจำกัดสิทธิ์ ถ้าถูก compromise จะได้กระทบแค่ส่วนที่ user นี้เข้าถึงได้เท่านั้น

> **ทำบนเครื่อง Server** ใน PowerShell Administrator

### ขั้นตอนที่ 1 — สร้าง User

```powershell
$password = Read-Host -AsSecureString "Enter password for deverloper"
New-LocalUser -Name "deverloper" -Password $password -FullName "SIT Deploy User" -Description "SSH deploy account"
```

| Option | ความหมาย |
|--------|----------|
| `-Name "deverloper"` | ชื่อ username ที่ใช้ตอน SSH เข้า เช่น `ssh deverloper@192.168.1.50` |
| `-Password` | password ของ user นี้บนเครื่อง Windows — ใช้ตอน login ด้วย password แต่ถ้าใช้ Key auth จะไม่ได้ใช้ตอน SSH |
| `-FullName` | ชื่อแสดงใน Windows user management |
| `-Description` | หมายเหตุบอกว่า account นี้ใช้ทำอะไร |

### ขั้นตอนที่ 2 — เพิ่มเข้า Group

```powershell
# เพิ่มเข้า Users group (สิทธิ์ขั้นต่ำ)
Add-LocalGroupMember -Group "Users" -Member "deverloper"

# เพิ่มเข้า Administrators เฉพาะถ้า deploy script ต้องการสิทธิ์สูง
# Add-LocalGroupMember -Group "Administrators" -Member "deverloper"
```

### ขั้นตอนที่ 3 — สร้าง Working Directory สำหรับ Deploy

```powershell
# สร้าง folder สำหรับวาง deploy artifacts
New-Item -ItemType Directory -Path "C:\sit-deploy" -Force

# ให้สิทธิ์ Modify แก่ user deverloper (อ่าน เขียน ลบได้ แต่ไม่ได้เป็นเจ้าของ)
$acl = Get-Acl "C:\sit-deploy"
$rule = New-Object System.Security.AccessControl.FileSystemAccessRule(
    "deverloper", "Modify", "ContainerInherit,ObjectInherit", "None", "Allow"
)
$acl.SetAccessRule($rule)
Set-Acl "C:\sit-deploy" $acl
```

| Permission | ความหมาย |
|------------|----------|
| `Modify` | อ่าน เขียน แก้ไข ลบไฟล์ได้ แต่ไม่สามารถเปลี่ยน permission ของ folder ได้ |
| `ContainerInherit` | sub-folder ที่สร้างใหม่ใน `C:\sit-deploy` จะได้รับ permission นี้ด้วย |
| `ObjectInherit` | ไฟล์ที่สร้างใหม่ใน `C:\sit-deploy` จะได้รับ permission นี้ด้วย |

### ตรวจสอบ User ที่สร้าง

```powershell
Get-LocalUser -Name "deverloper"
Get-LocalGroupMember -Group "Users"
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

Admin จะนำ public key ไปวางใน `C:\Users\deverloper\.ssh\authorized_keys` บน server (ดูข้อ 4)

---

### 7.4 ตั้งค่า SSH Config File (แนะนำ — ไม่ต้องพิมพ์ option ทุกครั้ง)

SSH Config File คือไฟล์ที่เก็บการตั้งค่าการเชื่อมต่อของแต่ละ server ไว้ล่วงหน้า แทนที่จะต้องพิมพ์ option ยาวๆ ทุกครั้ง เช่น `ssh -i ~/.ssh/sit_deploy_key -p 22 deverloper@192.168.1.50` ก็ย่อเหลือแค่ `ssh sit`

สร้างหรือแก้ไขไฟล์ที่ `C:\Users\<username>\.ssh\config` (ไม่มีนามสกุล):

```powershell
notepad "$env:USERPROFILE\.ssh\config"
```

วางเนื้อหาดังนี้:

```sshconfig
# SIT Server (ใน LAN)
Host sit
    HostName 192.168.1.50
    User deverloper
    Port 22
    IdentityFile C:\Users\<username>\.ssh\sit_deploy_key
    ServerAliveInterval 60
    ServerAliveCountMax 3

# SIT Server ผ่าน VPN (จากภายนอก)
Host sit-vpn
    HostName 192.168.1.50
    User deverloper
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
ssh deverloper@192.168.1.50

# Key login ระบุ identity file เอง
ssh -i ~/.ssh/sit_deploy_key deverloper@192.168.1.50

# ระบุ port ถ้าเปลี่ยนจาก 22
ssh -p 2222 -i ~/.ssh/sit_deploy_key deverloper@192.168.1.50
```

---

### 7.7 SFTP (upload/download files)

```bash
# ใช้ alias จาก SSH Config
sftp sit

# หรือพิมพ์เต็ม
sftp -i ~/.ssh/sit_deploy_key deverloper@192.168.1.50

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
scp -i ~/.ssh/sit_deploy_key ./app.zip deverloper@192.168.1.50:/sit-deploy/

# อัปโหลดทั้ง folder
scp -r -i ~/.ssh/sit_deploy_key ./dist/ deverloper@192.168.1.50:/sit-deploy/

# ดาวน์โหลดไฟล์จาก server
scp -i ~/.ssh/sit_deploy_key deverloper@192.168.1.50:/sit-deploy/logs/app.log ./

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

## 8. เพิ่ม Client User ใหม่

เมื่อมี Developer คนใหม่ต้องการ SSH เข้า Server มีขั้นตอนที่แต่ละฝ่ายต้องทำดังนี้

### Flow Overview

```
[Client — Developer ใหม่]           [Admin — ดูแล Server]
          │                                   │
  ① สร้าง SSH Key Pair                        │
          │                                   │
  ② ส่ง .pub ให้ Admin ─────────────────────► │
                                     ③ เพิ่ม Public Key
                                        ใน authorized_keys
                                              │
          │ ◄──────────── แจ้ง IP + username ─┤
          │                                   │
  ④ ตั้งค่า SSH Config                        │
          │                                   │
  ⑤ ทดสอบ SSH เข้า Server                    │
          │                                   │
  ⑥ แจ้ง Admin ว่าเข้าได้แล้ว ──────────────► │
                                     ⑦ (optional)
                                        ปิด PasswordAuth
                                        ถ้าทุก client ใช้ key แล้ว
```

---

### ① Client — สร้าง SSH Key Pair

สร้าง folder `.ssh` และ key pair บนเครื่องตัวเอง:

```powershell
# Windows
New-Item -ItemType Directory -Path "$env:USERPROFILE\.ssh" -Force

ssh-keygen -t ed25519 -C "yourname-sit" -f "$env:USERPROFILE\.ssh\sit_deploy_key"
```

```bash
# macOS / Linux
mkdir -p ~/.ssh && chmod 700 ~/.ssh

ssh-keygen -t ed25519 -C "yourname-sit" -f ~/.ssh/sit_deploy_key
```

ตั้ง passphrase เมื่อถามเสมอ — ถ้า private key หลุดออกไปคนอื่นจะใช้ไม่ได้โดยไม่รู้ passphrase

ไฟล์ที่ได้:

| ไฟล์ | ส่งให้ใคร |
|------|-----------|
| `sit_deploy_key` | เก็บไว้บนเครื่องตัวเอง **ห้ามแชร์** |
| `sit_deploy_key.pub` | ส่งให้ Admin |

---

### ② Client — ดู Public Key เพื่อส่งให้ Admin

```powershell
# Windows
Get-Content "$env:USERPROFILE\.ssh\sit_deploy_key.pub"
```

```bash
# macOS / Linux
cat ~/.ssh/sit_deploy_key.pub
```

ตัวอย่าง output ที่ได้:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx yourname-sit
```

Copy บรรทัดนี้ทั้งหมดส่งให้ Admin ผ่าน Slack / Teams / Email

---

### ③ Admin — เพิ่ม Public Key บน Server

เปิดไฟล์ `authorized_keys` บน Server:

```powershell
notepad C:\Users\deverloper\.ssh\authorized_keys
```

เพิ่ม public key ที่ได้รับต่อท้ายไฟล์ (คนละบรรทัด ถ้ามีหลาย client):

```
ssh-ed25519 AAAAC3NzaC1lZ... user1-sit
ssh-ed25519 AAAAC3NzaC1lZ... user2-sit       ← เพิ่มบรรทัดใหม่ที่นี่
```

บันทึกไฟล์แล้วตรวจสอบว่า permission ยังถูกต้อง:

```powershell
Get-Acl "C:\Users\deverloper\.ssh\authorized_keys" | Format-List
# ต้องเห็นแค่ deverloper และ SYSTEM เท่านั้น
```

> ไม่ต้อง restart sshd — OpenSSH อ่าน `authorized_keys` ทุกครั้งที่มี connection ใหม่

---

### ④ Client — ตั้งค่า SSH Config

สร้างหรือแก้ไข `~/.ssh/config` เพิ่ม Host entry:

```powershell
notepad "$env:USERPROFILE\.ssh\config"
```

```sshconfig
Host sit
    HostName 192.168.1.50
    User deverloper
    Port 22
    IdentityFile C:\Users\<username>\.ssh\sit_deploy_key
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

แทน `<username>` ด้วยชื่อ user Windows ของตัวเอง

---

### ⑤ Client — ทดสอบ SSH

```powershell
# ใช้ alias จาก config
ssh sit

# หรือพิมพ์เต็ม
ssh -i "$env:USERPROFILE\.ssh\sit_deploy_key" deverloper@192.168.1.50
```

ถ้าเข้าได้สำเร็จจะเห็น PowerShell prompt ของ Server หรือ Banner ที่ตั้งไว้

ถ้าพบปัญหาให้ดูที่ [Troubleshooting](#11-troubleshooting)

---

### ⑦ Admin — ลบ Public Key (กรณีลาออกหรือเปลี่ยนเครื่อง)

เปิด `authorized_keys` และลบบรรทัดของ user นั้นออก:

```powershell
notepad C:\Users\deverloper\.ssh\authorized_keys
```

ลบบรรทัดที่มี comment ตรงกับ user ที่ต้องการลบ เช่น `... username-sit` แล้ว Save

ไม่ต้อง restart sshd — มีผลทันทีในการ login ครั้งถัดไป

---

## 9. VPN สำหรับเข้าจากภายนอก

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
ssh -i ~/.ssh/sit_deploy_key deverloper@10.8.0.2
```

### Option C — ใช้ VPN ที่มีอยู่แล้วในบริษัท (Cisco AnyConnect, FortiClient, etc.)

1. เชื่อมต่อ VPN ด้วย client ของบริษัท
2. ตรวจสอบ IP ที่ได้รับจาก VPN (มักเป็น 10.x.x.x หรือ 172.16.x.x)
3. SSH ตรงไปยัง IP ของเครื่อง Server ใน LAN ได้เลย (ถ้า routing ถูกต้อง)

```bash
ssh deverloper@192.168.1.50
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
Get-Acl "C:\Users\deverloper\.ssh\authorized_keys" | Format-List

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
ssh -vvv deverloper@192.168.1.50
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
