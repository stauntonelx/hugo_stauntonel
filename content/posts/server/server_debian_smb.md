---
title: Samba server
date: 2025-12-27
draft: false
description: Samba server
categories:
  - server
tags:
  - Debian
  - samba
slug: server-debian-samba
toc: true
---

Installeer een smb op een debian trixie server.
Configureer samba mount op client

<!--more-->

## 1. software

Installeer de volgende software:

```
sudo apt install samba smbclient cifs-utils
```

## 2. configuratie

### Voorbereidingen

allereerst stoppen we de service.

```
sudo systemctl restart smbd nmbd
```

Daarna gaan we een gebruiker en mappenstructuur aanmaken.

```
sudo useradd -M -s /sbin/nologin smbuser
sudo usermod -aG sambashare smbuser
sudo mkdir -p /srv/samba/LinuxShare/
sudo chown -R smbuser:sambashare /srv/samba/
sudo chmod -R 2770 /srv/samba/
sudo smbpasswd -a smbuser
sudo smbpasswd -e smbuser
```

smbshare: groep die we gebruiken voor de share.
smbuser: systeem gebruiker (-M) zonder login shell
share map: /srv/samba met de juiste rechten (gebruiker en groep)
smppasswd:

- -a geef de gebruiker een smb wachtwoord

- -e activeer de gebruiker

### Global sessie

Open het config bestand met je favoriete editor

```
sudo systemct restart smbd nmbd
sudo nano /etc/samba/smb.conf
```

en geeft het de volgende inhoud:

```
[global]

   workgroup = WORKGROUP
   log file = /var/log/samba/log.%J
   max log size = 1000
   logging = file
   panic action = /usr/share/samba/panic-action %d
   server role = standalone server
   map to guest = Never
```

workgroup: alle computers moeten deel uitmaken van deze werkproep
logging: naar /var/log/samba lognaam log.%J  -> ip_nummer als extensie
panic action: als smb crasht vind je hier meer info
server role: standalone -> share gebonden aan gebruiker / wachtwoord
map to guest: Als wachtwoord niet juist is dan weiger toegang 

### Shares

```
#======================= Share Definitions =======================

[LinuxShare]
    comment = share voor linux data
    path = /srv/samba/LinuxShare
    read only = no
    guest ok = no
    writable = yes
    valid users = smbuser @sambashare
    write list = smbuser @sambashare
    create mask = 0664
    directory mask = 0775
    force create mode = 0664
    force directory mode = 2775
    force group = sambashare
    inherit permissions = yes
```

Check of alles in orde is met 

```
testparm -s
```

### Herstart

Nadat het opslaan van de config moet we de service herstarten.

```
sudo systemctl restart smbd nmbd
```

## 03 Mount op client

### Filemanager

Zelf gebruik ik thunar. In de adresbalk geef je dit in:

```
smb://dock03/linuxshare/
```

Geef daarna de smb credentials in van smbuser.

### rclone

Configureer rclone remote

```
rclone config
```

Kies voor smb en volg de instructie op het scherm.
Check daarna *~/.config/rclone/rclone.conf* die ziet er ongeveer zo uit:

```
[dock03_smb]
type = smb
host = dock03.home
user = smbuser
pass = ***************
```

Met dit commando raadpleeg je welke shares er zijn:

```
rclone lsd dock03_smb:
```

mount de share in terminal met:

```
rclone mount dock03_smb:/home/jack/Share/LinuxShare --daemon
```

Als je wilt kun je met systemd een opstart unit maken.
 
