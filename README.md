# homeserver
Setup:
OS: Arch Linux

-Pre Prep-

Download Arch Linux https://mirrors.urbanwave.co.za/archlinux/iso/2024.06.01/archlinux-x86_64.iso

Archinstall script
-minimal
-multilib
Username - serveradmin > sudo
Username - andyserver > normal
Username - lindieserver > normal
Username - tavianserver > normal

Additional packages:
- git
- vim
- ranger
- samba
- nftables
- openssh
- base-devel
- netdata
- apache
- motion
- cronie
- fail2ban
- python-pip
- python-virtualenv
- libffi
- synapse
- logwatch
- clamav

git packages:
- https://aur.archlinux.org/plex-media-server.git

Matrix setup:
Add user: sudo useradd -r -m -U -d /var/lib/synapse synapse

Configure synapse:
sudo su - synapse
virtualenv -p python3 synapse
source synapse/bin/activate
pip install matrix-synapse

Generate config:
synctl start
synctl stop

edit config:
vim /var/lib/synapse/homeserver.yaml

Start IM:
synctl start

Monitoring software:
sudo systemctl enable netdata
sudo systemctl start netdata

Samba config:
vim /etc/samba/smb.conf

Paste this:
[global]
    workgroup = WORKGROUP
    server string = Samba Server %v
    netbios name = PotNet
    security = user
    map to guest = bad user
    dns proxy = no

[Shared]
    path = /srv/samba/shared
    browsable = yes
    writable = yes
    guest ok = no
    read only = no
    create mask = 0777
    directory mask = 0777
    valid users = lindie, andy, tavian

create shared directory:
sudo mkdir -p /srv/samba/share
sudo chown -R nobody:nobody /srv/samba/share
sudo chmod -R 0775 /srv/samba/share

start services:
sudo systemctl enable smb nmb
sudo systemctl start smb nmb

Intranet:
sudo systemctl enable httpd
sudo systemctl start httpd

move webpages to: /srv/http/

Security cameras:

vim /etc/motion/motion.conf

Paste this:
daemon on
stream_localhost off
stream_port 8081
webcontrol_localhost off
webcontrol_port 8080

Media server:
sudo systemctl enable plexmediaserver
sudo systemctl start plexmediaserver

enable auto update:
sudo vim /usr/local/bin/system_update.sh

Paste this:
#!/bin/bash

sudo pacman -Syu --noconfirm
yay -Syu --noconfirm
sudo paccache -r
echo "System updated on $(date)" >> /var/log/system_update.log

terminal this:
sudo chmod +x /usr/local/bin/system_update.sh

enable daemon:
sudo systemctl enable cronie
sudo systemctl start cronie
sudo crontab -e
0 * * * * /usr/local/bin/system_update.sh

SSH:
vim /etc/ssh/sshd_config

Paste this:
PermitRootLogin no
PasswordAuthentication no
AllowUsers your_username

terminal this:
sudo systemctl restart sshd

fail2ban:
sudo pacman -S fail2ban
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

Clamav:
sudo freshclam
sudo systemctl enable clamav-freshclam
sudo systemctl start clamav-freshclam
sudo systemctl enable clamav-daemon
sudo systemctl start clamav-daemon

firewall:
vim /etc/nftables.conf

paste this:
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;

        # Allow established and related connections
        ct state established,related accept

        # Allow loopback interface
        iif lo accept

        # Allow ICMP
        ip protocol icmp accept

        # Allow SSH
        tcp dport 22 accept

        # Allow HTTP and HTTPS
        tcp dport {80, 443} accept

        # Allow Plex
        tcp dport 32400 accept

        # Allow Netdata
        tcp dport 19999 accept

        # Allow Samba
        tcp dport {137, 138, 139, 445} accept
        udp dport {137, 138, 139, 445} accept

        # Allow Motion
        tcp dport {8080, 8081} accept

        # Log and drop everything else
        log prefix "nftables: " counter drop
    }

    chain forward {
        type filter hook forward priority 0; policy drop;
    }

    chain output {
        type filter hook output priority 0; policy accept;
    }
}

Finish up by enabling the Quantum encryption script