# GLPI IT Asset Management Server on Ubuntu (Apache + MariaDB + PHP)

A home-lab project: installing and configuring **GLPI 12.0.1**, an open-source IT asset management and helpdesk platform, on an Ubuntu virtual machine running in VMware Workstation.

## Goal

Build a working GLPI server from scratch on the LAMP stack, and document the installation and the problems I solved along the way.

## Environment

| Component | Details |
|---|---|
| Hypervisor | VMware Workstation (host: Windows) |
| Guest OS | Ubuntu Server |
| Web server | Apache 2.4 |
| Database | MariaDB 11.8 |
| PHP | 8.5 |
| Application | GLPI 12.0.1 |
| Network | VMware NAT (VMnet8, subnet 192.168.136.0/24) |

## Installation steps

### 1. Install the stack

```bash
sudo apt update
sudo apt install apache2 mariadb-server php php-mysql php-curl php-gd php-intl \
  php-mbstring php-xml php-zip php-bz2 php-ldap php-opcache php-exif php-xmlrpc
```

### 2. Create the database and user

```bash
sudo mariadb
```

```sql
CREATE DATABASE glpi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'glpiuser'@'localhost' IDENTIFIED BY '<your-password>';
GRANT ALL PRIVILEGES ON glpi.* TO 'glpiuser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### 3. Download and place GLPI

```bash
cd /tmp
wget https://github.com/glpi-project/glpi/releases/download/12.0.1/glpi-12.0.1.tgz
tar -xzf glpi-12.0.1.tgz
sudo mv glpi /var/www/
sudo chown -R www-data:www-data /var/www/glpi
```

### 4. Configure the Apache virtual host

GLPI 11 and later must be served from the `public` folder.

`/etc/apache2/sites-available/glpi.conf`:

```apache
<VirtualHost *:80>
    DocumentRoot /var/www/glpi/public
    <Directory /var/www/glpi/public>
        Require all granted
        RewriteEngine On
        RewriteCond %{HTTP:Authorization} ^(.+)$
        RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
        RewriteCond %{REQUEST_FILENAME} !-f
        RewriteRule ^(.*)$ index.php [QSA,L]
    </Directory>
</VirtualHost>
```

```bash
sudo a2enmod rewrite
sudo a2dissite 000-default.conf
sudo a2ensite glpi.conf
sudo apache2ctl configtest      # expect: Syntax OK
sudo systemctl restart apache2
```

### 5. Run the web installer

Open `http://<vm-ip>/` in a browser, pass the requirements check, enter the database details (`localhost`, `glpiuser`, the password, database `glpi`) and finish the wizard.

## Troubleshooting log

| Problem | Cause | Fix |
|---|---|---|
| `Invalid command 'Require'` / `DocumentRoot does not exist` | Typos in `glpi.conf` (`Reqiure`, `/Var/www`). Linux paths are case-sensitive. | Corrected the file and re-ran `apache2ctl configtest` until it showed `Syntax OK`. |
| `RewriteCond: bad argument line` | The second argument `!-f` was missing or mistyped. | Fixed the line to `RewriteCond %{REQUEST_FILENAME} !-f`. |
| Browser: `ERR_CONNECTION_TIMED_OUT`; ping to the VM went out to the internet | "Connect a host virtual adapter to this network" was unchecked for VMnet8, so Windows had no route to the VM subnet. | Enabled the host virtual adapter in VMware's Virtual Network Editor. |
| Installer showed failed required checks (`curl`, `gd`) and a warning for `ldap` | Missing PHP extensions. | `sudo apt install php-curl php-gd php-ldap`, then restarted Apache. |
| `Access denied for user 'glpiuser'@'localhost'` | The stored database password did not match the one entered. | Reset it with `ALTER USER ... IDENTIFIED BY ...`, re-granted privileges and flushed them. |

## What I learned

- Installing and configuring a LAMP stack on Ubuntu from the command line
- Apache virtual hosts, `mod_rewrite`, and testing config with `apache2ctl configtest`
- MariaDB user and privilege management
- File ownership and permissions for web applications (`www-data`)
- VMware NAT networking and why a host virtual adapter is needed to reach a guest VM
- Reading error messages methodically and fixing one cause at a time

## Next steps

- Install the GLPI inventory agent and register a test machine
- Enable HTTPS with a self-signed certificate
- Set up LDAP / Active Directory authentication
- Automate backups of the `glpi` database

## Security note

Default GLPI accounts and the database password were changed after installation. No real credentials are stored in this repository.

## Author

Zakaria El Jarni, IT / Networking student
