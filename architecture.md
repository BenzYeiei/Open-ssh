# OpenSSH Architecture — Server & Client

คู่มือนี้แสดงโครงสร้างของ component ทั้งฝั่ง Server และ Client สำหรับ SIT Deploy Environment

---

## Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        EXTERNAL NETWORK                             │
│                                                                     │
│   [Developer นอกออฟฟิศ]                                            │
│         │                                                           │
│         │ VPN Tunnel (OpenVPN :1194 / WireGuard :51820)            │
│         ▼                                                           │
│   [VPN Gateway / Router]                                            │
│         │                                                           │
│         │ Route → LAN 192.168.1.0/24                               │
└─────────┼───────────────────────────────────────────────────────────┘
          │
┌─────────┼───────────────────────────────────────────────────────────┐
│         │               OFFICE LAN 192.168.1.0/24                  │
│         │                                                           │
│   [Developer ในออฟฟิศ] ──────────────────────────────────┐         │
│                                                           │         │
│                                              SSH :22      │         │
│                                                           ▼         │
│                                        ┌──────────────────────────┐ │
│                                        │   Windows SSH Server     │ │
│                                        │   IP: 192.168.1.50       │ │
│                                        └──────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Server Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                   Windows SSH Server                        │
│                   IP: 192.168.1.50                          │
│                                                             │
│  ┌─────────────────┐    ┌──────────────────────────────┐   │
│  │  Windows        │    │  Windows Firewall             │   │
│  │  Services       │    │                               │   │
│  │                 │    │  Rule: OpenSSH-Server-In-TCP  │   │
│  │  ┌───────────┐  │    │  Port: 22                     │   │
│  │  │  sshd     │◄─┼────┤  RemoteAddress:               │   │
│  │  │ (OpenSSH  │  │    │    192.168.1.0/24 (LAN)       │   │
│  │  │  Server)  │  │    │    10.8.0.0/24   (VPN)        │   │
│  │  │           │  │    └──────────────────────────────┘   │
│  │  │ Port: 22  │  │                                        │
│  │  │ Auto-start│  │    ┌──────────────────────────────┐   │
│  │  └───────────┘  │    │  Default Shell                │   │
│  └─────────────────┘    │                               │   │
│                         │  Registry:                    │   │
│  ┌─────────────────┐    │  HKLM:\SOFTWARE\OpenSSH       │   │
│  │  sshd_config    │    │  DefaultShell = powershell.exe│   │
│  │                 │    └──────────────────────────────┘   │
│  │  Path:          │                                        │
│  │  C:\ProgramData │    ┌──────────────────────────────┐   │
│  │  \ssh\          │    │  Users                        │   │
│  │  sshd_config    │    │                               │   │
│  │                 │    │  ┌────────────┐               │   │
│  │  Port 22        │    │  │ deverloper │ (deploy user) │   │
│  │  PubkeyAuth yes │    │  │ Group:Users│               │   │
│  │  PasswordAuth no│    │  └────────────┘               │   │
│  │  AllowUsers:    │    │  ┌─────────────────┐          │   │
│  │   deverloper    │    │  │  administrator  │           │   │
│  │   administrator │    │  └─────────────────┘          │   │
│  └─────────────────┘    └──────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Filesystem                                         │   │
│  │                                                     │   │
│  │  C:\ProgramData\ssh\                                │   │
│  │  ├── sshd_config          ← main config             │   │
│  │  └── banner.txt           ← login warning (optional)│   │
│  │                                                     │   │
│  │  C:\Users\deverloper\                               │   │
│  │  └── .ssh\                                          │   │
│  │      └── authorized_keys  ← public keys ของ clients │   │
│  │          Permission: deverloper + SYSTEM only       │   │
│  │                                                     │   │
│  │  C:\sit-deploy\           ← deploy working dir      │   │
│  │      Permission: deverloper (Modify)                │   │
│  │      ContainerInherit + ObjectInherit               │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Server File Map

| Path | ประเภท | หน้าที่ |
|------|--------|---------|
| `C:\ProgramData\ssh\sshd_config` | Config | ตั้งค่าหลักของ OpenSSH Server |
| `C:\ProgramData\ssh\banner.txt` | Text | ข้อความแสดงก่อน login prompt |
| `C:\Users\deverloper\.ssh\authorized_keys` | Key Store | รวม Public Key ของทุก Client ที่อนุญาต |
| `C:\sit-deploy\` | Directory | Deploy artifacts, scripts, logs |
| `HKLM:\SOFTWARE\OpenSSH` | Registry | ตั้งค่า Default Shell |

### sshd_config Key Settings

```
Port 22
AddressFamily inet
ListenAddress 0.0.0.0

PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PasswordAuthentication no          ← ปิดหลัง setup key เสร็จ

LoginGraceTime 30
MaxAuthTries 5
MaxSessions 10

ClientAliveInterval 60
ClientAliveCountMax 3

Subsystem sftp sftp-server.exe
AllowUsers deverloper administrator
```

---

## Client Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                   Developer Machine (Client)                │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  SSH Client Tools                                   │   │
│  │                                                     │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │   │
│  │  │   ssh    │  │   scp    │  │      sftp        │  │   │
│  │  │          │  │          │  │                  │  │   │
│  │  │ Terminal │  │ One-shot │  │ Interactive      │  │   │
│  │  │ session  │  │ transfer │  │ file transfer    │  │   │
│  │  └────┬─────┘  └────┬─────┘  └────────┬─────────┘  │   │
│  │       └─────────────┴─────────────────┘             │   │
│  │                     │                               │   │
│  │                     ▼                               │   │
│  │  ┌──────────────────────────────────────────────┐  │   │
│  │  │  ~/.ssh/                                     │  │   │
│  │  │                                              │  │   │
│  │  │  config              ← SSH aliases & options │  │   │
│  │  │  sit_deploy_key      ← Private Key (ห้ามแชร์)│  │   │
│  │  │  sit_deploy_key.pub  ← Public Key (ส่ง admin)│  │   │
│  │  │  known_hosts         ← Server fingerprints   │  │   │
│  │  │                                              │  │   │
│  │  │  Permission: Owner only (FullControl)        │  │   │
│  │  └──────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  VPN Client (สำหรับเชื่อมต่อจากภายนอก)              │   │
│  │                                                     │   │
│  │  Option A: OpenVPN Client (.ovpn config)            │   │
│  │  Option B: WireGuard Client (wg0.conf)              │   │
│  │  Option C: Cisco AnyConnect / FortiClient           │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Client File Map

| Path | ประเภท | หน้าที่ |
|------|--------|---------|
| `~/.ssh/config` | Config | Aliases และ default options ของแต่ละ host |
| `~/.ssh/sit_deploy_key` | Private Key | ใช้ authenticate กับ server — ห้ามแชร์ |
| `~/.ssh/sit_deploy_key.pub` | Public Key | ส่งให้ server admin วางใน `authorized_keys` |
| `~/.ssh/known_hosts` | Key Store | Fingerprint ของ server ที่เคยเชื่อมต่อ |

### SSH Config File

```sshconfig
# ใน LAN (เครื่องในออฟฟิศ)
Host sit
    HostName 192.168.1.50
    User deverloper
    Port 22
    IdentityFile C:\Users\<username>\.ssh\sit_deploy_key
    ServerAliveInterval 60
    ServerAliveCountMax 3

# ผ่าน VPN (จากภายนอก)
Host sit-vpn
    HostName 192.168.1.50
    User deverloper
    Port 22
    IdentityFile C:\Users\<username>\.ssh\sit_deploy_key
    ServerAliveInterval 60
    ServerAliveCountMax 3
```

---

## Authentication Flow

```
Client                                    Server
  │                                          │
  │──── TCP SYN → Port 22 ──────────────────►│
  │                                          │  ตรวจ Firewall Rule
  │◄─── TCP SYN-ACK ────────────────────────│  (RemoteAddress whitelist)
  │                                          │
  │──── SSH Protocol Version ───────────────►│
  │◄─── SSH Protocol Version ───────────────│
  │                                          │
  │──── Key Exchange (Diffie-Hellman) ──────►│
  │◄─── Session Key established ────────────│
  │                                          │
  │──── Auth Request (publickey) ───────────►│
  │                                          │  อ่าน authorized_keys
  │◄─── Challenge (random data) ────────────│  ตรวจ permission ของไฟล์
  │                                          │
  │  [sign challenge with private key]       │
  │──── Signature ──────────────────────────►│
  │                                          │  verify signature ด้วย public key
  │◄─── Auth Success ───────────────────────│
  │                                          │
  │◄─── Shell / Command Session ────────────│
  │                                          │
```

---

## Key Pair Relationship

```
┌──────────────────────┐              ┌──────────────────────┐
│      CLIENT          │              │       SERVER         │
│                      │              │                      │
│  ~/.ssh/             │  ส่งครั้งเดียว│  C:\Users\deverloper\│
│  sit_deploy_key      │   (setup)    │  .ssh\authorized_keys│
│  sit_deploy_key.pub  │─────────────►│                      │
│                      │  .pub only   │  ssh-ed25519 AAA...  │
│  Private Key:        │              │  (line ต่อ 1 key)    │
│  - เก็บในเครื่อง     │              │                      │
│  - ห้ามแชร์          │              │  Permission:         │
│  - ใช้ sign challenge│              │  deverloper + SYSTEM │
│                      │              │  only                │
└──────────────────────┘              └──────────────────────┘
```

---

## Network Ports & Protocols

| Component | Protocol | Port | Direction |
|-----------|----------|------|-----------|
| OpenSSH Server | TCP | 22 | Inbound (Server) |
| OpenVPN | UDP | 1194 | Inbound (VPN Gateway) |
| WireGuard | UDP | 51820 | Inbound (VPN Gateway) |
| SFTP (via SSH) | TCP | 22 | ใช้ช่องทางเดิมกับ SSH |
| SCP (via SSH) | TCP | 22 | ใช้ช่องทางเดิมกับ SSH |
