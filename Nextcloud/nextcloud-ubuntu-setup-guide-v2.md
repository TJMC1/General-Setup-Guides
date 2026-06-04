# Nextcloud on Ubuntu Server — Full Setup Guide
## (with Self-Signed HTTPS → NGINX Proxy Manager)

---

## Overview

This guide installs Nextcloud on a **bare-metal Ubuntu Server 24.04 LTS VM** in Proxmox, configures it with a self-signed certificate for internal HTTPS, and sets it up to sit behind **NGINX Proxy Manager (NPM)** which handles the public-facing SSL from Let's Encrypt.

**Architecture:**

```
Internet → NGINX Proxy Manager (public SSL / Let's Encrypt)
                    ↓ (internal HTTPS, self-signed)
           Nextcloud VM (Ubuntu 24.04)
                    ↓
           Mounted SMB shares / ZFS data disk
```

---

## Part 1 — Create the Ubuntu VM in Proxmox

1. In Proxmox, click **Create VM**
2. Recommended specs:
   - **OS:** Ubuntu Server 24.04 LTS ISO
   - **CPU:** 2 cores
   - **RAM:** 2–4 GB
   - **Disk:** 32 GB (OS/app only — data goes on a separate disk, see Part 4)
   - **Network:** VirtIO, assign a static IP via your router/OPNsense DHCP reservation

3. Install Ubuntu Server — during setup:
   - Choose **minimized install**
   - Enable **OpenSSH server** when prompted
   - Skip snaps at the end

4. Once booted, SSH in and update:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Part 2 — Install Nextcloud (Bare Metal LAMP Stack)

### Install dependencies

```bash
sudo apt install -y apache2 mariadb-server libapache2-mod-php   php php-gd php-mysql php-curl php-mbstring php-intl php-gmp   php-bcmath php-xml php-imagick php-zip php-bz2 php-ldap   php-smbclient smbclient php-apcu redis-server php-redis   unzip wget curl bzip2
```

> `php-ldap`, `php-smbclient`, and `bzip2` are all included here. `bzip2` is required to extract the Nextcloud archive and is not included in Ubuntu's minimised install.

### Configure MariaDB

```bash
sudo mysql_secure_installation
# Answer Y to all prompts, set a strong root password

sudo mysql -u root -p
```

Inside the MariaDB shell:

```sql
CREATE DATABASE nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'ncuser'@'localhost' IDENTIFIED BY 'your_strong_password';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'ncuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Download and install Nextcloud

```bash
cd /tmp
wget https://download.nextcloud.com/server/releases/latest.tar.bz2
sudo tar -xjf latest.tar.bz2
sudo mv nextcloud /var/www/
sudo chown -R www-data:www-data /var/www/nextcloud
sudo chmod -R 755 /var/www/nextcloud
```

---

## Part 3 — Configure PHP

### Set values in the Apache PHP config

Open `/etc/php/8.3/apache2/php.ini` and set these values (note: do NOT edit the CLI config at `/etc/php/8.3/cli/php.ini` — Nextcloud reads the Apache one):

```bash
sudo sed -i 's/^memory_limit.*/memory_limit = 512M/' /etc/php/8.3/apache2/php.ini
sudo sed -i 's/^upload_max_filesize.*/upload_max_filesize = 10G/' /etc/php/8.3/apache2/php.ini
sudo sed -i 's/^post_max_size.*/post_max_size = 10G/' /etc/php/8.3/apache2/php.ini
sudo sed -i 's/^max_execution_time.*/max_execution_time = 360/' /etc/php/8.3/apache2/php.ini
sudo sed -i 's|^;*date.timezone.*|date.timezone = Europe/London|' /etc/php/8.3/apache2/php.ini
```

### Set OPcache values

> OPcache settings are in a separate config file, not `php.ini`. Edit the Apache OPcache config directly — uncomment and set these values:

```bash
sudo nano /etc/php/8.3/apache2/conf.d/10-opcache.ini
```

Set the following (remove the leading semicolons if present):

```ini
opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=64
opcache.max_accelerated_files=100000
opcache.revalidate_freq=1
opcache.save_comments=1
opcache.validate_timestamps=0
```

### Restart Apache

```bash
sudo systemctl restart apache2
```

### Verify

```bash
php -i | grep -E "memory_limit|upload_max|post_max|max_execution|timezone|opcache.interned_strings_buffer|opcache.memory_consumption|opcache.max_accelerated_files"
```

> Note: `php -i` and `php --ini` always show the CLI config, not Apache's. Use the Nextcloud admin overview page to confirm warnings have cleared.

---

## Part 4 — Add a Data Disk (Separate from OS)

For data disks **2 TB or larger**, you must use GPT partitioning — not MBR/fdisk.

In Proxmox, **shut down the VM**, go to **Hardware → Add → Hard Disk**, set your desired size, then start the VM again.

```bash
# Install gdisk (required for 2TB+ disks)
sudo apt install -y gdisk

# Find the new disk (usually /dev/sdb)
sudo fdisk -l

# Partition with GPT
sudo gdisk /dev/sdb
```

Inside gdisk:
```
o        # create new GPT partition table → type y to confirm
n        # new partition
          # Enter (default partition number: 1)
          # Enter (default first sector)
          # Enter (default last sector — full disk)
          # Enter (default type: Linux filesystem)
w        # write and exit → type y to confirm
```

```bash
# Format
sudo mkfs.ext4 /dev/sdb1

# Create mount point
sudo mkdir -p /mnt/ncdata

# Get UUID
sudo blkid /dev/sdb1   # copy the UUID value

# Add to fstab for persistence
sudo nano /etc/fstab
```

Add this line (replace UUID):

```
UUID=xxxx-xxxx  /mnt/ncdata  ext4  defaults  0  2
```

```bash
sudo mount -a
sudo chown -R www-data:www-data /mnt/ncdata
```

---

## Part 5 — Configure Apache with Self-Signed HTTPS

### Enable required Apache modules

```bash
sudo a2enmod rewrite headers env dir mime ssl
```

### Generate a self-signed certificate

```bash
sudo mkdir -p /etc/ssl/nextcloud
sudo openssl req -x509 -nodes -days 3650 -newkey rsa:4096   -keyout /etc/ssl/nextcloud/nextcloud.key   -out /etc/ssl/nextcloud/nextcloud.crt   -subj "/CN=nextcloud.local/O=Homelab/C=GB"
```

### Create the Apache vhost

```bash
sudo nano /etc/apache2/sites-available/nextcloud-ssl.conf
```

Paste the following:

```apache
<VirtualHost *:443>
    ServerName nextcloud.local
    DocumentRoot /var/www/nextcloud

    SSLEngine on
    SSLCertificateFile /etc/ssl/nextcloud/nextcloud.crt
    SSLCertificateKeyFile /etc/ssl/nextcloud/nextcloud.key

    <Directory /var/www/nextcloud>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews

        <IfModule mod_dav.c>
            Dav off
        </IfModule>
    </Directory>

    Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-XSS-Protection "1; mode=block"
    Header always set Referrer-Policy "no-referrer"

    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
</VirtualHost>

<VirtualHost *:80>
    ServerName nextcloud.local
    Redirect permanent / https://nextcloud.local/
</VirtualHost>
```

### Enable the site and disable the default

```bash
sudo a2ensite nextcloud-ssl.conf
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
```

---

## Part 6 — Run the Nextcloud Web Installer

Browse to `https://<VM-IP>` (accept the self-signed cert warning — expected).

Fill in the setup form:

| Field | Value |
|---|---|
| Admin username | your choice |
| Admin password | strong password |
| Data folder | `/mnt/ncdata` |
| Database | MySQL/MariaDB |
| DB user | `ncuser` |
| DB password | the password you set |
| DB name | `nextcloud` |
| DB host | `localhost` |

Click **Install** and wait ~2 minutes.

### Post-install config

```bash
sudo nano /var/www/nextcloud/config/config.php
```

Add/edit these values inside the config array:

```php
'trusted_domains' =>
array (
  0 => '<VM-IP>',
  1 => 'your.public.domain.com',
),
'overwrite.cli.url' => 'https://your.public.domain.com',
'overwriteprotocol' => 'https',
'overwritehost' => 'your.public.domain.com',
'trusted_proxies' =>
array (
  0 => '<NPM-VM-IP>',
),
'forwarded_for_headers' =>
array (
  0 => 'HTTP_X_FORWARDED_FOR',
),
'memcache.local' => '\OC\Memcache\APCu',
'memcache.locking' => '\OC\Memcache\Redis',
'redis' =>
array (
  'host' => 'localhost',
  'port' => 6379,
),
```

---

## Part 7 — Post-Install Fixes

### Maintenance window

```bash
sudo -u www-data php /var/www/nextcloud/occ config:system:set maintenance_window_start --type=integer --value=1
```

### Phone region

```bash
sudo -u www-data php /var/www/nextcloud/occ config:system:set default_phone_region --value="GB"
```

### Mimetype migration

```bash
sudo -u www-data php /var/www/nextcloud/occ maintenance:repair --include-expensive
```

### Imagick SVG support

```bash
sudo apt install -y libmagickcore-6.q16-6-extra
sudo systemctl restart apache2
```

### Clear file locks (if needed)

```bash
sudo -u www-data php /var/www/nextcloud/occ maintenance:mode --on
sudo mysql -u root -p nextcloud -e "DELETE FROM oc_file_locks WHERE 1;"
sudo -u www-data php /var/www/nextcloud/occ maintenance:mode --off
```

---

## Part 8 — Configure NGINX Proxy Manager

In NPM, add a new **Proxy Host**:

| Setting | Value |
|---|---|
| Domain Names | `your.public.domain.com` |
| Scheme | `https` |
| Forward Hostname/IP | `<Nextcloud VM IP>` |
| Forward Port | `443` |
| Cache Assets | Off |
| Block Common Exploits | On |
| Websockets Support | On |
| SSL — Let's Encrypt | Enable, force SSL |

> Under the **SSL** tab, set **Verify SSL Certificate → OFF** (Nextcloud uses a self-signed cert internally). NPM still forwards over HTTPS — it just won't verify the internal cert chain.

Add this in the **Advanced** tab:

```nginx
client_max_body_size 10G;
proxy_read_timeout 600s;
proxy_send_timeout 600s;

location /.well-known/carddav {
    return 301 $scheme://$host/remote.php/dav;
}
location /.well-known/caldav {
    return 301 $scheme://$host/remote.php/dav;
}
```

---

## Part 9 — SMB External Storage

Enable the **External Storage** app: Admin → Apps → search "External storage" → Enable.

### Mount SMB share at OS level

```bash
sudo apt install cifs-utils
sudo nano /etc/samba/smb-credentials
```

```
username=your_smb_user
password=your_smb_password
domain=WORKGROUP
```

```bash
sudo chmod 600 /etc/samba/smb-credentials
sudo mkdir -p /mnt/myshare
sudo nano /etc/fstab
```

Add:

```
//192.168.x.x/sharename  /mnt/myshare  cifs  credentials=/etc/samba/smb-credentials,iocharset=utf8,uid=www-data,gid=www-data,_netdev  0  0
```

```bash
sudo mount -a
```

In Nextcloud → Admin → External Storage:
- **Storage type:** Local
- **Configuration:** `/mnt/myshare`

---

## Part 10 — LDAP Integration

Enable the **LDAP / AD Integration** app: Admin → Apps → Enable.

Go to **Admin → LDAP/AD Integration**:

| Field | Value |
|---|---|
| Server | `ldap://192.168.x.x` or `ldaps://192.168.x.x` |
| Port | `389` (LDAP) or `636` (LDAPS) |
| Bind DN | e.g. `cn=admin,dc=example,dc=local` |
| Password | bind account password |
| Base DN | `dc=example,dc=local` |

### Adding a second NIC for AD VLAN access

If your AD is on a separate VLAN, add a second NIC in Proxmox (Hardware → Add → Network Device) and configure it with DHCP in Netplan:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

```yaml
network:
  version: 2
  ethernets:
    ens18:                          # existing NIC — leave untouched
      addresses: [10.100.100.10/24]
      routes:
        - to: default
          via: 10.100.100.1
    ens19:                          # AD VLAN NIC
      dhcp4: true
```

```bash
sudo netplan apply
```

> Do NOT add a second default route on `ens19` — only the primary NIC needs the gateway.

### For LDAPS with a private CA

```bash
sudo cp your-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

### Test connectivity before configuring in Nextcloud

```bash
ping -c 3 <DC-IP>
nc -vz <DC-IP> 389
nc -vz <DC-IP> 636
```

---

## Part 11 — Background Jobs (Cron)

```bash
sudo crontab -u www-data -e
```

Add:

```
*/5 * * * * php -f /var/www/nextcloud/cron.php
```

In Nextcloud: **Admin → Basic Settings → Background Jobs → select Cron**.

---

## Part 12 — Keeping Things Updated

### OS updates

```bash
sudo apt update && sudo apt upgrade -y
```

### Nextcloud updates

Via web UI: **Admin → Overview → Update**, or via CLI:

```bash
sudo -u www-data php /var/www/nextcloud/updater/updater.phar
```

> Always **snapshot the VM in Proxmox** before running a major Nextcloud version upgrade.

---

## Quick Reference

| Item | Value |
|---|---|
| Nextcloud (internal) | `https://<VM-IP>` |
| Nextcloud (public) | `https://your.public.domain.com` |
| SSH | `ssh ubuntu@<VM-IP>` |
| Data directory | `/mnt/ncdata` |
| SMB mounts | `/mnt/<sharename>` |
| Nextcloud config | `/var/www/nextcloud/config/config.php` |
| Apache PHP config | `/etc/php/8.3/apache2/php.ini` |
| Apache OPcache config | `/etc/php/8.3/apache2/conf.d/10-opcache.ini` |
| Apache logs | `/var/log/apache2/nextcloud_error.log` |
