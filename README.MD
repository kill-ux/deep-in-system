# deep-in-system

## Overview

`deep-in-system` is a project for learning how to configure a Linux server with the most common services used in production environments, such as SSH, FTP, MySQL, and WordPress. The goal is to understand how to set up, secure, and manage a server from scratch.

---
ova file is in https://mega.nz/file/kMUhgYZR#qL8-qNQKS3O8nSYkqlYB3g-87LdPC3mkyS1XD7D9mxc

---

## 1. Virtual Machine Setup

### Disk Partitioning

The 30GB disk is divided into 4 partitions to manage storage and access carefully:

| Partition | Size | Purpose                                                       |
| --------- | ---- | ------------------------------------------------------------- |
| swap      | 4G   | Handles RAM overflow. Not for files — acts as slow backup RAM |
| / (root)  | 15G  | The main partition. Contains OS, services, and dependencies   |
| /home     | 5G   | Each local user has their own space here                      |
| /backup   | 6G   | Dedicated partition for storing backups                       |

Separating partitions prevents one service from consuming all disk space and crashing the system.

### Installation Details

- **OS:** Ubuntu Server 24.04 LTS
- **Hostname:** `muboutoub-host`
- **Username:** `muboutoub`
- **Filesystem:** ext4 (stable, default Linux filesystem)

---

## 2. Network Configuration

A static IP is required because services like WordPress and SSH must always be reachable at the same address. With DHCP the IP can change on reboot, breaking all connections.

### Tool: Netplan

Configuration file: `/etc/netplan/50-cloud-init.yaml`

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: false
      addresses:
        - 10.1.8.16/16
      routes:
        - to: default
          via: 10.1.0.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

Apply changes:
```bash
sudo netplan apply
```

- **Static IP:** `10.1.8.16`
- **DNS:** `8.8.8.8` (Google DNS — translates domain names to IPs)
- **Adapter:** Bridge mode (VM appears as a real machine on the network)

---

## 3. Security

### SSH Hardening

Configuration file: `/etc/ssh/sshd_config`

| Change                         | Value | Reason                                       |
| ------------------------------ | ----- | -------------------------------------------- |
| Port                           | 2222  | Reduces automated attacks on default port 22 |
| PermitRootLogin                | no    | Prevents direct root access via SSH          |
| PasswordAuthentication (luffy) | no    | Key-only auth for admin user                 |
| PubkeyAuthentication (zoro)    | no    | Password-only auth for normal user           |

> **Note (Ubuntu 24.04):** Port changes require restarting `ssh.socket` not just `ssh.service`:
```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

Config changes (PermitRootLogin etc.) only need:
```bash
sudo systemctl restart ssh
```

### Firewall (UFW)

UFW (Uncomplicated Firewall) checks every incoming request and allows or denies it based on rules.

```bash
sudo ufw allow 2222/tcp   # SSH
sudo ufw allow 80/tcp     # HTTP (WordPress)
sudo ufw allow 21/tcp     # FTP control
sudo ufw allow 40000:50000/tcp  # FTP passive data transfer
sudo ufw enable
```

All other incoming ports are blocked by default.

---

## 4. User Management

| User      | Auth Method    | Sudo | Purpose                          |
| --------- | -------------- | ---- | -------------------------------- |
| muboutoub | password       | yes  | Main admin user                  |
| luffy     | SSH public key | yes  | Admin user (key auth only)       |
| zoro      | password       | no   | Normal user (no elevated access) |
| nami      | password       | no   | FTP-only user                    |

### Password vs SSH Key Authentication

- **Password:** User types credentials. Vulnerable to brute force attacks.
- **SSH Key:** Server holds public key. Client proves identity by signing a challenge with private key. Private key never travels over the network — more secure.

### Creating a user with SSH key auth and sudo

```bash
# On server
sudo adduser username
sudo usermod -aG sudo username

# On host machine
ssh-keygen -t ed25519 -C "username"
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 username@10.1.8.16
```

> The SSH key for `luffy` is stored at `~/.ssh/id_ed25519` on the host machine.

---

## 5. FTP Server (vsftpd)

**vsftpd** (Very Secure FTP Daemon) handles file transfers. Note: standard FTP is not encrypted — use SFTP or FTPS for production.

### nami user restrictions

- Shell set to `/sbin/nologin` → blocks SSH access
- `/sbin/nologin` added to `/etc/shells` → allows FTP via PAM
- `chroot_local_user=YES` → jails nami inside `/backup` (she cannot navigate outside)
- `local_root=/backup` → sets her root to `/backup`
- `write_enable=NO` → read-only access

### Key vsftpd settings (`/etc/vsftpd.conf`)

```ini
anonymous_enable=NO
local_enable=YES
write_enable=NO
chroot_local_user=YES
local_root=/backup
allow_writeable_chroot=YES
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=50000
userlist_enable=YES
userlist_file=/etc/vsftpd.userlist
userlist_deny=NO
```

### Active vs Passive FTP

- **Active:** Server connects back to client — blocked by client firewalls
- **Passive:** Client initiates both connections — works through firewalls ✅

---

## 6. MySQL Database

### Security measures

- `bind-address = 127.0.0.1` → MySQL only listens locally, no remote connections
- Root uses `auth_socket` → only accessible via `sudo mysql`, not remotely
- `wpuser@localhost` → WordPress uses a dedicated user with minimum privileges

### WordPress database setup

```sql
CREATE DATABASE wordpress;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
```

`wpuser` has access **only** to the `wordpress` database — not root, not other databases.

---

## 7. WordPress

WordPress is installed in Apache's web root directory `/var/www/html/` so it's accessible at `http://10.1.8.16/`.

### Stack

```
Browser → Apache (HTTP server) → PHP (processes code) → MySQL (data)
```

### wp-config.php protection

`wp-config.php` contains database credentials and must never be publicly accessible.

Added to `/var/www/html/.htaccess`:
```apache
<files wp-config.php>
    deny from all
</files>
```

Visiting `http://10.1.8.16/wp-config.php` returns **403 Forbidden**.

`AllowOverride All` is set in `/etc/apache2/sites-available/000-default.conf` to allow Apache to read `.htaccess` rules.

---

## 8. Backup System

### Backup script (`/usr/local/bin/backup.sh`)

```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_FILE="/backup/wordpress_$DATE.tar.gz"
START=$(date +%T)

mysqldump --defaults-file=/etc/mysql/backup.cnf wordpress | gzip > $BACKUP_FILE

echo "[$START] Backup successful: $BACKUP_FILE" >> /var/log/backup.log
```

- Creates a compressed database dump with the date in the filename
- Logs each backup to `/var/log/backup.log`
- Backup files are accessible via nami's FTP account

### Cron job

```bash
sudo crontab -e
```

```
0 0 * * * /usr/local/bin/backup.sh
```

Runs every day at **00:00 (midnight)**. Cron format: `minute hour day month weekday`.

---

## 9. Open Ports Summary

| Port        | Protocol | Service     | Reason                   |
| ----------- | -------- | ----------- | ------------------------ |
| 2222        | TCP      | SSH         | Remote server management |
| 80          | TCP      | HTTP        | WordPress website        |
| 21          | TCP      | FTP         | File transfer control    |
| 40000-50000 | TCP      | FTP Passive | File transfer data       |

---

## Credentials

>  For audit purposes only. Never expose credentials in real projects!

| User      | Password                      | Auth Method | Sudo |
| --------- | ----------------------------- | ----------- | ---- |
| muboutoub | `    ` (4 spaces)             | password    | yes  |
| luffy     | SSH key (see id_ed25519 file) | public key  | yes  |
| zoro      | `    ` (4 spaces)             | password    | no   |
| nami      | `    ` (4 spaces)             | FTP only    | no   |

### MySQL
| User   | Password |
| ------ | -------- |
| wpuser | password |

### WordPress Admin
| User  | Password |
| ----- | -------- |
| admin | admin    |

### SSH Key for luffy
Private key is stored at: `~/.ssh/id_ed25519` on host machine

## Resources

1. [Ubuntu Server Guide](https://ubuntu.com/server/docs)
2. [Netplan Documentation](https://netplan.io/)
3. [vsftpd Documentation](https://security.appspot.com/vsftpd.html)
4. [WordPress Installation Guide](https://wordpress.org/support/article/how-to-install-wordpress/)
5. [MySQL Documentation](https://dev.mysql.com/doc/)