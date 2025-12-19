# XAMPP Configuration Guide: PHP 8.5, Apache, MySQL, and phpMyAdmin

This comprehensive guide covers setting up and configuring XAMPP with PHP 8.5, Apache on custom ports, MySQL with SSL/TLS encryption, and phpMyAdmin for secure database management. This setup ensures modern security, compatibility, and performance for local development.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Installation and Setup](#installation-and-setup)
- [PHP 8.5 Configuration](#php-85-configuration)
- [Apache Configuration](#apache-configuration)
- [MySQL Configuration](#mysql-configuration)
- [SSL/TLS Setup for MySQL](#ssltls-setup-for-mysql)
- [phpMyAdmin Configuration](#phpmyadmin-configuration)
- [Testing and Verification](#testing-and-verification)
- [Troubleshooting](#troubleshooting)
- [Security Best Practices](#security-best-practices)
- [Backup and Restore](#backup-and-restore)

## Prerequisites
- Windows 10/11 (64-bit)
- XAMPP with Apache, MySQL (MariaDB), and phpMyAdmin
- Administrative privileges for service management and certificate installation
- Basic knowledge of command-line operations and web server configuration

## Installation and Setup
1. Download and install XAMPP from the official website.
2. Launch XAMPP Control Panel as Administrator.
3. Install PHP 8.5:
   - Download PHP 8.5.0 from php.net.
   - Extract to `d:\xampp\php_8.5.0`.
   - Copy `php.ini` from `d:\xampp\php\php.ini` and adjust paths.
4. Update XAMPP's Apache to use PHP 8.5 by editing `d:\xampp\apache\conf\httpd.conf`:
   ```
   LoadModule php_module "d:\xampp\php_8.5.0\php8apache2_4.dll"
   PHPIniDir "d:\xampp\php_8.5.0"
   ```

## PHP 8.5 Configuration
Edit `d:\xampp\php_8.5.0\php.ini`:
- Set `session.save_path = "D:\xampp\xampp\tmp"`
- Set `extension_dir = "ext"`
- Enable extensions: `extension=openssl`, `extension=mysqli`, `extension=pdo_mysql`
- Adjust memory: `memory_limit = 256M`
- Enable error reporting for development: `error_reporting = E_ALL`

Restart Apache after changes.

## Apache Configuration
1. Change default ports to avoid conflicts:
   - Edit `d:\xampp\apache\conf\httpd.conf`:
     ```
     Listen 8080
     ServerName localhost:8080
     ```
   - Edit `d:\xampp\apache\conf\extra\httpd-ssl.conf`:
     ```
     Listen 8443
     ```
2. Enable SSL module in `httpd.conf`:
   ```
   LoadModule ssl_module modules/mod_ssl.so
   Include conf/extra/httpd-ssl.conf
   ```
3. Generate self-signed certificates:
   ```
   cd d:\xampp\apache
   makecert.bat
   ```
4. Configure VirtualHost for phpMyAdmin in `d:\xampp\apache\conf\extra\phpmyadmin-ssl.conf`:
   ```apache
   <VirtualHost *:80>
       ServerName localhost
       Redirect permanent / https://localhost:8080/
   </VirtualHost>

   <VirtualHost *:8080>
       ServerName localhost
       DocumentRoot "D:/xampp/xampp/htdocs"
       Alias /phpmyadmin "D:/xampp/xampp/phpMyAdmin"
       <Directory "D:/xampp/xampp/phpMyAdmin">
           Options Indexes FollowSymLinks
           AllowOverride All
           Require all granted
           Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
           Header always set X-Content-Type-Options "nosniff"
           Header always set X-Frame-Options "SAMEORIGIN"
       </Directory>
   </VirtualHost>

   <VirtualHost *:8443>
       ServerName localhost
       DocumentRoot "D:/xampp/xampp/htdocs"
       Alias /phpmyadmin "D:/xampp/xampp/phpMyAdmin"
       <Directory "D:/xampp/xampp/phpMyAdmin">
           Options Indexes FollowSymLinks
           AllowOverride All
           Require all granted
           SSLEngine on
           SSLCertificateFile "D:/xampp/xampp/apache/crt/localhost/ssl/localhost.crt"
           SSLCertificateKeyFile "D:/xampp/xampp/apache/crt/localhost/ssl/localhost.key"
           SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
           SSLCipherSuite HIGH:!aNULL:!MD5:!3DES
           SSLHonorCipherOrder on
       </Directory>
   </VirtualHost>
   ```

Restart Apache.

## MySQL Configuration
Edit `d:\xampp\mysql\bin\my.ini`:
- Set `port=3306`
- Set `datadir="D:/xampp/xampp/mysql/data"`
- Enable InnoDB: `innodb_file_per_table=1`
- Set `key_buffer_size=16M`
- Enable performance schema if needed

Restart MySQL.

## SSL/TLS Setup for MySQL
1. Generate CA and certificates:
   ```
   cd d:\xampp\mysql\data
   openssl genrsa 2048 > ca-key.pem
   openssl req -new -x509 -nodes -days 365000 -key ca-key.pem -out ca.pem -subj "/C=US/ST=State/L=City/O=Development/CN=CA"
   openssl req -newkey rsa:2048 -days 365000 -nodes -keyout server-key.pem -out server-req.pem -subj "/C=US/ST=State/L=City/O=Development/CN=127.0.0.1"
   openssl rsa -in server-key.pem -out server-key.pem
   openssl x509 -req -in server-req.pem -days 365000 -CA ca.pem -CAkey ca-key.pem -set_serial 01 -out server-cert.pem
   ```
2. Update `my.ini`:
   ```
   ssl-ca = D:/xampp/xampp/mysql/data/ca.pem
   ssl-cert = D:/xampp/xampp/mysql/data/server-cert.pem
   ssl-key = D:/xampp/xampp/mysql/data/server-key.pem
   ```
3. Restart MySQL.

## phpMyAdmin Configuration
Edit `d:\xampp\phpMyAdmin\config.inc.php`:
```php
$cfg['blowfish_secret'] = 'your_secret_key';
$cfg['ForceSSL'] = false;
$cfg['PmaAbsoluteUri'] = 'http://localhost:8080/phpmyadmin/';
$cfg['Servers'][$i]['host'] = '127.0.0.1';
$cfg['Servers'][$i]['port'] = 3306;
$cfg['Servers'][$i]['connect_type'] = 'tcp';
$cfg['Servers'][$i]['auth_type'] = 'cookie';
$cfg['Servers'][$i]['user'] = 'root';
$cfg['Servers'][$i]['password'] = 'your_password';
$cfg['Servers'][$i]['ssl'] = true;
$cfg['Servers'][$i]['ssl_ca'] = 'D:/xampp/xampp/mysql/data/ca.pem';
$cfg['Servers'][$i]['ssl_verify'] = false;
$cfg['Servers'][$i]['pmadb'] = 'phpmyadmin';
// Add other pma tables...
```

Create phpMyAdmin configuration storage:
- Import `d:\xampp\phpMyAdmin\sql\create_tables.sql` in phpMyAdmin.

## Testing and Verification
- Access http://localhost:8080/ for Apache.
- Access http://localhost:8080/phpmyadmin/ for phpMyAdmin.
- Test MySQL SSL: `mysql --ssl-ca="D:/xampp/xampp/mysql/data/ca.pem" -u root -p -e "STATUS"`
- Check PHP: Create `test.php` with `<?php phpinfo(); ?>`

## Troubleshooting
- **MySQL won't start**: Check `mysql_error.log` for errors.
- **phpMyAdmin login fails**: Verify SSL settings and certificate paths.
- **Port conflicts**: Ensure no other services use 8080/8443/3306.
- **SSL errors**: Regenerate certificates or set `ssl_verify = false`.
- **phpMyAdmin configuration storage**: Run the create_tables.sql script.

## Security Best Practices
- Use strong passwords for MySQL root.
- Keep certificates updated; regenerate before expiration.
- Enable `require_secure_transport = ON` in `my.ini` for production-like security.
- Monitor logs for anomalies.
- Use HTTPS for all connections.

## Backup and Restore
- Backup databases: `mysqldump -u root -p --all-databases > backup.sql`
- Restore: `mysql -u root -p < backup.sql`
- For data files, stop services before copying.

This setup provides a secure, modern development environment. For production, use CA-signed certificates and harden further.