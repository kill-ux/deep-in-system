# deep-in-system

## Overview

`deep-in-system` is a project for learning how to configure a Linux server with the most common services used in production environments, such as SSH, FTP, MySQL, and WordPress. The goal is to understand how to set up, secure, and manage a server from scratch.

> The VM file (.ova) is available at: https://mega.nz/file/kMUhgYZR#qL8-qNQKS3O8nSYkqlYB3g-87LdPC3mkyS1XD7D9mxc

---

## 1. Virtual Machine Setup

### Concepts Learned
- A **virtual machine** is a software-based computer running inside your real machine using a hypervisor (VirtualBox).
- **Disk partitioning** separates storage into isolated sections so one service can't consume all disk space and crash the system.
- **ext4** is the default Linux filesystem — it manages how files are stored and retrieved on disk.
- **swap** is not a real filesystem — it's raw space used when RAM is full, acting as slow backup memory.

### Disk Partitioning

The 30GB disk is divided into 4 partitions:

| Partition | Size | Purpose                                                       |
| --------- | ---- | ------------------------------------------------------------- |
| swap      | 4G   | Handles RAM overflow. Not for files — acts as slow backup RAM |
| / (root)  | 15G  | The main partition. Contains OS, services, and dependencies   |
| /home     | 5G   | Each local user has their own space here                      |
| /backup   | 6G   | Dedicated partition for storing backups                       |

### Installation Details

- **OS:** Ubuntu Server 24.04 LTS
- **Hostname:** `muboutoub-host`
- **Username:** `muboutoub`
- **Filesystem:** ext4

### Commands Used

```bash
lsblk                               # list all block devices and partitions
hostname                            # show current hostname
sudo hostnamectl set-hostname name  # change hostname
whoami                              # show current user
```

---

## 2. Network Configuration

### Concepts Learned
- **DHCP** (Dynamic Host Configuration Protocol) automatically assigns IP addresses — they can change on reboot.
- **Static IP** is required for servers so services like SSH and WordPress are always reachable at the same address.
- **Netplan** is Ubuntu's network configuration tool — reads YAML files and applies network settings.
- **Gateway** is the door out of your network — wrong gateway = no internet.
- **DNS** translates domain names like `google.com` into IP addresses.
- `/16` netmask means 16 bits for network, 16 bits for hosts.
- **Bridge adapter** makes the VM appear as a real machine on your network.

### Configuration

File: `/etc/netplan/50-cloud-init.yaml`

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

### Commands Used

```bash
ip a                    # show all network interfaces and IPs
ip route                # show routing table and gateway
ping -c 5 google.com    # test internet connectivity
sudo netplan apply      # apply netplan configuration
```

---

## 3. Security

### Concepts Learned
- **SSH** (Secure Shell) is an encrypted protocol for remote server access.
- Changing the default port 22 reduces automated bot attacks.
- **PermitRootLogin no** prevents attackers from targeting root directly.
- **UFW** (Uncomplicated Firewall) controls incoming/outgoing traffic.
- **TCP** guarantees delivery — used for SSH, HTTP, FTP.
- **UDP** is faster but no guarantee — used for DNS, streaming.
- In **Ubuntu 24.04**, SSH port is controlled by `ssh.socket` — different from older versions where only `sshd_config` was needed.

### SSH Installation

```bash
sudo apt install openssh-server
```

### SSH Configuration

File: `/etc/ssh/sshd_config`

```
Port 2222
PermitRootLogin no

Match User luffy
    PasswordAuthentication no

Match User zoro
    PubkeyAuthentication no
```

### UFW Configuration

```bash
sudo ufw allow 2222/tcp         # SSH
sudo ufw allow 80/tcp           # HTTP (WordPress)
sudo ufw allow 21/tcp           # FTP control
sudo ufw allow 40000:50000/tcp  # FTP passive data
sudo ufw enable
```

### Commands Used

```bash
sudo systemctl daemon-reload        # reload systemd after config changes
sudo systemctl restart ssh.socket   # apply port changes (Ubuntu 24.04)
sudo systemctl restart ssh          # apply config changes
sudo sshd -t                        # test SSH config for errors
sudo ss -tlnp                       # show listening ports and services
sudo ufw status                     # show firewall rules
sudo systemctl is-enabled ssh       # check if service starts on boot
```

---

## 4. User Management

### Concepts Learned
- **Password auth** — user types credentials. Vulnerable to brute force.
- **SSH key auth** — server holds public key, client signs a challenge with private key. More secure.
- **ed25519** — modern elliptic curve algorithm, faster and more secure than RSA.
- **`-aG` flag** — append to group. Without `-a`, usermod replaces all existing groups.
- **`/sbin/nologin`** — fake shell that immediately closes connections, blocking SSH.

### User Table

| User      | Auth Method    | Sudo | Purpose                    |
| --------- | -------------- | ---- | -------------------------- |
| muboutoub | password       | yes  | Main admin user            |
| luffy     | SSH public key | yes  | Admin user (key auth only) |
| zoro      | password       | no   | Normal user                |
| nami      | password       | no   | FTP-only user              |

### Installation & Setup

```bash
# Create user
sudo adduser username

# Add to sudo group
sudo usermod -aG sudo username

# Restrict to nologin shell (FTP-only users)
sudo usermod -s /sbin/nologin username

# Generate SSH key (on host machine)
ssh-keygen -t ed25519 -C "username"

# Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 username@10.1.8.16
```

### Commands Used

```bash
groups username             # show user's groups
id username                 # show user ID and groups
sudo usermod -aG sudo user  # add user to sudo group
sudo usermod -s /bin user   # change user shell
ssh-keygen -t ed25519       # generate SSH key pair
ssh-copy-id                 # copy public key to server
```

---

## 5. FTP Server (vsftpd)

### Concepts Learned
- **FTP** (File Transfer Protocol) — port 21 for control, port 20 for data. Not encrypted.
- **vsftpd** (Very Secure FTP Daemon) — most common FTP server on Linux.
- **chroot** (change root) — jails a user inside a directory. They see it as `/` and cannot navigate above it.
- **Active FTP** — server connects back to client. Blocked by client firewalls.
- **Passive FTP** — client initiates both connections. Works through firewalls. Needs open port range on server.
- **PAM** (Pluggable Authentication Modules) — Linux authentication framework. Adding `/sbin/nologin` to `/etc/shells` allows PAM to accept FTP logins while blocking SSH.

### Installation

```bash
sudo apt install vsftpd
```

### Configuration

File: `/etc/vsftpd.conf`

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

### nami user setup

```bash
sudo adduser nami
sudo usermod -s /sbin/nologin nami
echo "/sbin/nologin" | sudo tee -a /etc/shells
echo "nami" | sudo tee /etc/vsftpd.userlist
sudo systemctl restart vsftpd
```

### Commands Used

```bash
sudo apt install vsftpd         # install FTP server
sudo systemctl restart vsftpd   # restart FTP service
ftp -p 10.1.8.16                # connect via FTP (passive mode)
cat /etc/shells                 # list valid shells
```

---

## 6. MySQL Database

### Concepts Learned
- **MySQL** is a relational database — stores data in structured tables.
- **MySQL root** controls only MySQL — separate from Linux root.
- **auth_socket** — Ubuntu's default MySQL root auth. Only accessible via `sudo mysql`, never remotely.
- **bind-address = 127.0.0.1** — MySQL only listens on localhost, blocking remote connections.
- **Principle of least privilege** — `wpuser` gets only the access it needs.
- **mysqldump** — exports database as SQL instructions to safely backup while server is running.

### Installation

```bash
sudo apt install mysql-server
sudo mysql_secure_installation
```

### Configuration

File: `/etc/mysql/mysql.conf.d/mysqld.cnf`
```
bind-address = 127.0.0.1
```

### WordPress Database Setup

```sql
CREATE DATABASE wordpress;
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'StrongPassword';
GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
GRANT PROCESS ON *.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
```

### Commands Used

```bash
sudo apt install mysql-server       # install MySQL
sudo mysql                          # connect as root via socket
mysql -u wpuser -p                  # connect as wpuser
sudo systemctl restart mysql        # restart MySQL
SHOW DATABASES;                     # list databases
SHOW GRANTS FOR 'user'@'host';      # show user privileges
SELECT user, host FROM mysql.user;  # list MySQL users
```

---

## 7. WordPress

### Concepts Learned
- **WordPress** is a PHP-based CMS that makes building websites easier.
- **Apache** is an HTTP web server — receives browser requests and serves files.
- **PHP** is the language WordPress is written in.
- **php-mysql** is a bridge between PHP and MySQL.
- **`/var/www/html`** is Apache's default web root — files here are publicly accessible.
- **www-data** is Apache's user — owns web files for security.
- **AllowOverride All** tells Apache to respect `.htaccess` files.
- **`.htaccess`** — per-directory Apache config used to protect sensitive files.

### Installation

```bash
# Install Apache + PHP
sudo apt install apache2 php php-mysql php-curl php-gd php-mbstring php-xml php-zip

# Download WordPress
cd /tmp
wget https://wordpress.org/latest.tar.gz
tar -xzf latest.tar.gz
sudo mv wordpress/* /var/www/html/

# Set permissions
sudo chown -R www-data:www-data /var/www/html/
sudo chmod -R 755 /var/www/html/

# Enable rewrite module
sudo a2enmod rewrite
sudo systemctl restart apache2
```

### wp-config.php Protection

File: `/var/www/html/.htaccess`
```apache
<files wp-config.php>
    deny from all
</files>
```

File: `/etc/apache2/sites-available/000-default.conf`
```apache
<Directory /var/www/html>
    AllowOverride All
    Require all granted
</Directory>
```

Visiting `http://10.1.8.16/wp-config.php` returns **403 Forbidden**.

### Commands Used

```bash
sudo apt install apache2            # install web server
sudo a2enmod rewrite                # enable rewrite module
sudo systemctl restart apache2      # restart Apache
wget https://wordpress.org/latest.tar.gz  # download WordPress
tar -xzf file.tar.gz                # extract compressed archive
sudo chown -R www-data:www-data /path  # set ownership recursively
sudo chmod -R 755 /path             # set permissions recursively
```

---

## 8. Backup System

### Concepts Learned
- **cron** is Linux's task scheduler — runs commands automatically at specified times.
- **Cron format:** `minute hour day-of-month month day-of-week command`
- **`0 0 * * *`** means every day at midnight.
- **mysqldump** exports a database as SQL — safer than copying raw files, server stays running.
- **gzip** compresses the backup to save disk space.
- Storing credentials in a config file prevents password exposure in process lists.

### Backup Script

File: `/usr/local/bin/backup.sh`

```bash
#!/bin/bash
DATE=$(date +%Y-%m-%d_%H-%M-%S)
BACKUP_FILE="/backup/wordpress_$DATE.tar.gz"
START=$(date +%T)

mysqldump --defaults-file=/etc/mysql/backup.cnf wordpress | gzip > $BACKUP_FILE

echo "[$START] Backup successful: $BACKUP_FILE" >> /var/log/backup.log
```

MySQL credentials file: `/etc/mysql/backup.cnf`
```ini
[client]
user=wpuser
password=StrongPassword
```

```bash
sudo chmod 600 /etc/mysql/backup.cnf    # secure credentials
sudo chmod +x /usr/local/bin/backup.sh  # make executable
```

### Cron Job

```bash
sudo crontab -e
```

```
0 0 * * * /usr/local/bin/backup.sh
```

### Commands Used

```bash
sudo crontab -e             # edit root's cron jobs
sudo crontab -l             # list root's cron jobs
sudo systemctl status cron  # check cron is running
mysqldump                   # export database
date +%Y-%m-%d              # get formatted date
chmod +x script.sh          # make script executable
```

---

## 9. Open Ports Summary

| Port        | Protocol | Service     | Reason                   |
| ----------- | -------- | ----------- | ------------------------ |
| 2222        | TCP      | SSH         | Remote server management |
| 80          | TCP      | HTTP        | WordPress website        |
| 21          | TCP      | FTP         | File transfer control    |
| 40000-50000 | TCP      | FTP Passive | File transfer data       |

---

## 10. Credentials

> ⚠️ For audit purposes only. Never expose credentials in real projects!

| User      | Password | Auth Method | Sudo |
| --------- | -------- | ----------- | ---- |
| muboutoub | 4 spaces | password    | yes  |
| luffy     | SSH key  | public key  | yes  |
| zoro      | 4 spaces | password    | no   |
| nami      | 4 spaces | FTP only    | no   |

### MySQL

| User   | Password |
| ------ | -------- |
| wpuser | password |

### WordPress Admin

| User  | Password |
| ----- | -------- |
| admin | admin    |

### SSH Key for luffy
Private key stored at `~/.ssh/id_ed25519` on host machine (`chmod 600`).

---

## Resources

1. [Ubuntu Server Guide](https://ubuntu.com/server/docs)
2. [Netplan Documentation](https://netplan.io/)
3. [vsftpd Documentation](https://security.appspot.com/vsftpd.html)
4. [WordPress Installation Guide](https://wordpress.org/support/article/how-to-install-wordpress/)
5. [MySQL Documentation](https://dev.mysql.com/doc/)
6. [UFW Documentation](https://help.ubuntu.com/community/UFW)