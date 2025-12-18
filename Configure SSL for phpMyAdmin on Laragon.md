# SSL Configuration Guidance Manual for phpMyAdmin on Laragon

## Table of Contents

- [Document Control](#document-control)
- [Executive Summary](#executive-summary)
- [1. Architecture Overview](#1-architecture-overview)
  - [1.1 Dual-Layer Encryption Model](#11-dual-layer-encryption-model)
  - [1.2 Key Components](#12-key-components)
- [2. Pre-Configuration Checklist](#2-pre-configuration-checklist)
  - [2.1 System Requirements](#21-system-requirements)
  - [2.2 Directory Structure Verification](#22-directory-structure-verification)
- [3. HTTPS Configuration for Web Interface](#3-https-configuration-for-web-interface)
  - [3.1 Generate SSL Certificates for Apache](#31-generate-ssl-certificates-for-apache)
  - [3.2 Configure Apache for HTTPS](#32-configure-apache-for-https)
  - [3.3 Configure phpMyAdmin for HTTPS](#33-configure-phpmyadmin-for-https)
- [4. MySQL SSL Configuration](#4-mysql-ssl-configuration)
  - [4.1 Generate MySQL-Compatible SSL Certificates](#41-generate-mysql-compatible-ssl-certificates)
  - [4.2 Configure MySQL Server for SSL](#42-configure-mysql-server-for-ssl)
  - [4.3 Verify MySQL SSL Configuration](#43-verify-mysql-ssl-configuration)
- [5. Configuration Storage Setup](#5-configuration-storage-setup)
  - [5.1 Create phpMyAdmin Configuration Database](#51-create-phpmyadmin-configuration-database)
- [6. Troubleshooting Guide](#6-troubleshooting-guide)
  - [6.1 Common Issues and Solutions](#61-common-issues-and-solutions)
  - [6.2 Diagnostic Tools](#62-diagnostic-tools)
- [7. Maintenance Procedures](#7-maintenance-procedures)
  - [7.1 Certificate Renewal Schedule](#71-certificate-renewal-schedule)
  - [7.2 Backup Procedures](#72-backup-procedures)
  - [7.3 Monitoring and Logging](#73-monitoring-and-logging)
- [8. Security Best Practices](#8-security-best-practices)
  - [8.1 Development Environment](#81-development-environment)
  - [8.2 Production Considerations](#82-production-considerations)
- [9. Appendices](#9-appendices)
  - [Appendix A: Quick Reference Commands](#appendix-a-quick-reference-commands)
  - [Appendix B: Configuration File Locations](#appendix-b-configuration-file-locations)
  - [Appendix C: Useful Resources](#appendix-c-useful-resources)
- [Document Approval](#document-approval)

---

## Document Control
- **Document Title**: SSL Configuration Guidance for phpMyAdmin on Laragon
- **Version**: 1.0
- **Date**: December 2025
- **Applicable Environment**: Windows with Laragon, phpMyAdmin 6, MySQL 9.1
- **Target Audience**: System Administrators, Developers, DevOps Engineers

---

## Executive Summary

This guidance provides comprehensive instructions for configuring SSL/TLS encryption in a Laragon-based development environment, covering both HTTPS for web interface security and MySQL SSL for database connection encryption. The document addresses common configuration challenges and provides practical solutions for local development environments.

## 1. Architecture Overview

### 1.1 Dual-Layer Encryption Model
```
┌──────────┐     HTTPS     ┌──────────┐     MySQL SSL     ┌──────────┐
│  Browser │──────────────▶│  Apache  │─────────────────▶│  MySQL   │
│          │◀──────────────│ (PHP)    │◀─────────────────│  Server  │
└──────────┘   Encrypted   └──────────┘    Encrypted     └──────────┘
```

### 1.2 Key Components
- **Web Layer SSL**: Browser ↔ Apache (using HTTPS)
- **Database Layer SSL**: PHP/phpMyAdmin ↔ MySQL Server
- **Certificate Authority**: Self-signed certificates for local development

## 2. Pre-Configuration Checklist

### 2.1 System Requirements
- [ ] Laragon installed (Latest version recommended)
- [ ] phpMyAdmin 6 configured in `D:\laragon\etc\apps\phpmyadmin6\`
- [ ] MySQL 9.1+ running on default port 3306
- [ ] Administrative access to Windows system
- [ ] OpenSSL available in PATH

### 2.2 Directory Structure Verification
```
D:\laragon\
├── bin\
│   ├── apache\httpd-2.4.65-...\
│   ├── mysql\mysql-9.1.0-winx64\
│   └── php\php-8.4.5\
├── etc\
│   ├── apps\
│   │   ├── phpmyadmin6\
│   │   │   ├── config.inc.php
│   │   │   └── src\
│   │   └── ssl\                    ← SSL certificates directory
│   ├── apache2\sites-enabled\
│   └── ssl\
└── data\
    └── mysql-9.1\                  ← MySQL data with certificates
```

## 3. HTTPS Configuration for Web Interface

### 3.1 Generate SSL Certificates for Apache

#### 3.1.1 Automated Certificate Generation Script
Create `Generate-LocalhostCert.ps1`:

```powershell
# Localhost SSL Certificate Generator
# Run as Administrator

param(
    [string]$Password = "[YOUR_PASSWORD]"
)

# Configuration
$basePath = "D:\laragon"
$sslDir = "$basePath\etc\apps\ssl"
$pfxFile = "$sslDir\localhost.pfx"
$crtFile = "$sslDir\localhost.crt"
$keyFile = "$sslDir\localhost.key"

# Create directory if not exists
if (-not (Test-Path $sslDir)) {
    New-Item -ItemType Directory -Path $sslDir -Force | Out-Null
    Write-Host "Created SSL directory: $sslDir"
}

# Step 1: Generate self-signed certificate
Write-Host "Generating self-signed certificate for localhost..." -ForegroundColor Cyan

$certParams = @{
    Subject = "CN=localhost"
    TextExtension = @("2.5.29.17={text}DNS=localhost&IPAddress=127.0.0.1&IPAddress=::1")
    CertStoreLocation = "Cert:\LocalMachine\My"
    KeyAlgorithm = "RSA"
    KeyLength = 2048
    HashAlgorithm = "SHA256"
    NotAfter = (Get-Date).AddYears(5)
}

$cert = New-SelfSignedCertificate @certParams

Write-Host "✓ Certificate created in Windows Certificate Store" -ForegroundColor Green

# Step 2: Export to PFX
Write-Host "Exporting certificate to PFX format..." -ForegroundColor Cyan

$securePassword = ConvertTo-SecureString -String $Password -Force -AsPlainText
Export-PfxCertificate -Cert $cert -FilePath $pfxFile -Password $securePassword

Write-Host "✓ PFX exported to: $pfxFile" -ForegroundColor Green

# Step 3: Extract CRT and KEY using OpenSSL
Write-Host "Extracting CRT and KEY files..." -ForegroundColor Cyan

# Verify OpenSSL is available
if (-not (Get-Command openssl -ErrorAction SilentlyContinue)) {
    Write-Host "OpenSSL not found in PATH. Adding Laragon's OpenSSL..." -ForegroundColor Yellow
    $env:Path += ";$basePath\bin\apache\httpd-2.4.65-250724-Win64-VS17\bin"
}

# Extract private key
openssl pkcs12 -in $pfxFile -nocerts -out $keyFile -nodes -password pass:$Password 2>$null
if ($LASTEXITCODE -eq 0) {
    Write-Host "✓ Private key extracted: $keyFile" -ForegroundColor Green
} else {
    Write-Host "✗ Failed to extract private key" -ForegroundColor Red
    exit 1
}

# Extract certificate
openssl pkcs12 -in $pfxFile -clcerts -nokeys -out $crtFile -password pass:$Password 2>$null
if ($LASTEXITCODE -eq 0) {
    Write-Host "✓ Certificate extracted: $crtFile" -ForegroundColor Green
} else {
    Write-Host "✗ Failed to extract certificate" -ForegroundColor Red
    exit 1
}

# Step 4: Verify files
Write-Host "`nVerification:" -ForegroundColor Cyan
Write-Host "--------------"

if (Test-Path $crtFile) {
    $crtInfo = openssl x509 -in $crtFile -text -noout 2>$null | Select-String "Subject:" | Select-Object -First 1
    Write-Host "Certificate Subject: $crtInfo" -ForegroundColor Green
}

if (Test-Path $keyFile) {
    $keySize = (Get-Item $keyFile).Length / 1KB
    Write-Host "Key file size: $keySize KB" -ForegroundColor Green
}

Write-Host "`n✅ SSL certificate generation complete!" -ForegroundColor Green
Write-Host "Files available in: $sslDir" -ForegroundColor Yellow
Write-Host "Certificate password: $Password" -ForegroundColor Magenta
```

#### 3.1.2 Execution Instructions
```powershell
# Run PowerShell as Administrator
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
.\Generate-LocalhostCert.ps1

# Alternative with custom password
.\Generate-LocalhostCert.ps1 -Password "YourSecurePassword123!"
```

### 3.2 Configure Apache for HTTPS

#### 3.2.1 Enable SSL Module
Edit `D:\laragon\etc\apache2\httpd.conf`:
```apache
# Uncomment or add these lines
LoadModule ssl_module modules/mod_ssl.so
LoadModule socache_shmcb_module modules/mod_socache_shmcb.so
```

#### 3.2.2 Create Virtual Host Configuration
Create `D:\laragon\etc\apache2\sites-enabled\phpmyadmin-ssl.conf`:

```apache
<VirtualHost *:80>
    ServerName localhost
    Redirect permanent / https://localhost/
</VirtualHost>

<VirtualHost *:443>
    ServerName localhost
    ServerAdmin webmaster@localhost
    
    # Document Root for phpMyAdmin 6
    DocumentRoot "D:/laragon/etc/apps/phpmyadmin6/public"
    
    # Directory configuration
    <Directory "D:/laragon/etc/apps/phpmyadmin6/public">
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
        
        # Security headers
        Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
        Header always set X-Content-Type-Options "nosniff"
        Header always set X-Frame-Options "SAMEORIGIN"
    </Directory>
    
    # SSL Configuration
    SSLEngine on
    SSLCertificateFile "D:/laragon/etc/apps/ssl/localhost.crt"
    SSLCertificateKeyFile "D:/laragon/etc/apps/ssl/localhost.key"
    
    # SSL Protocols (Modern secure configuration)
    SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite HIGH:!aNULL:!MD5:!3DES
    SSLHonorCipherOrder on
    
    # Logging
    ErrorLog "D:/laragon/logs/apache/phpmyadmin-ssl-error.log"
    CustomLog "D:/laragon/logs/apache/phpmyadmin-ssl-access.log" combined
    
    # PHP configuration via proxy
    <FilesMatch \.php$>
        SetHandler "proxy:fcgi://127.0.0.1:9000"
    </FilesMatch>
</VirtualHost>
```

#### 3.2.3 Apply Configuration
```bash
# Restart Laragon services
# Method 1: Using Laragon GUI
# Right-click Laragon tray icon → "Restart All"

# Method 2: Command line
D:\laragon\laragon.exe restart
```

### 3.3 Configure phpMyAdmin for HTTPS

Edit `D:\laragon\etc\apps\phpmyadmin6\config.inc.php`:

```php
<?php
declare(strict_types=1);

/* ==================== SERVER CONFIGURATION ==================== */

/**
 * Encryption key for cookie authentication (REQUIRED)
 * Must be exactly 32 hexadecimal characters
 */
$cfg['blowfish_secret'] = '0164ff3896c2d9220a5eab9847f63d77';

/**
 * Force HTTPS connections
 */
$cfg['ForceSSL'] = true;
$cfg['PmaAbsoluteUri'] = 'https://localhost/phpmyadmin6/';

/**
 * Hide connection errors in production
 */
$cfg['Servers'][$i]['hide_connection_errors'] = false;

/* ==================== DATABASE CONNECTION ==================== */

/**
 * Primary server configuration
 */
$i = 1;  // Explicit server index

$cfg['Servers'][$i]['auth_type']     = 'cookie';      // Recommended for security
$cfg['Servers'][$i]['host']          = '127.0.0.1';   // Local MySQL server
$cfg['Servers'][$i]['port']          = '3306';        // Default MySQL port
$cfg['Servers'][$i]['connect_type']  = 'tcp';         // Connection type
$cfg['Servers'][$i]['extension']     = 'mysqli';      // MySQL extension
$cfg['Servers'][$i]['compress']      = false;         // Disable for local
$cfg['Servers'][$i]['AllowNoPassword'] = false;       // Security: require password

/**
 * SSL Configuration for MySQL connection
 * For local development, SSL verification can be disabled
 */
$cfg['Servers'][$i]['ssl']           = true;          // Enable SSL
$cfg['Servers'][$i]['ssl_verify']    = false;         // Disable cert verification for self-signed
// $cfg['Servers'][$i]['ssl_ca']     = 'D:/laragon/etc/ssl/ca.pem';      // Optional
// $cfg['Servers'][$i]['ssl_cert']   = 'D:/laragon/etc/ssl/client-cert.pem';
// $cfg['Servers'][$i]['ssl_key']    = 'D:/laragon/etc/ssl/client-key.pem';

/* ==================== CONFIGURATION STORAGE ==================== */

/**
 * phpMyAdmin configuration storage (advanced features)
 * Uncomment after creating the database
 */
$cfg['Servers'][$i]['controluser']  = 'pma';
$cfg['Servers'][$i]['controlpass']  = [YOUR_PASSWORD]';
$cfg['Servers'][$i]['pmadb']        = 'phpmyadmin';

/**
 * Essential tables for basic functionality
 */
$cfg['Servers'][$i]['bookmarktable']      = 'pma__bookmark';
$cfg['Servers'][$i]['relation']           = 'pma__relation';
$cfg['Servers'][$i]['table_info']         = 'pma__table_info';
$cfg['Servers'][$i]['table_coords']       = 'pma__table_coords';
$cfg['Servers'][$i]['pdf_pages']          = 'pma__pdf_pages';

/**
 * Optional tables (enable as needed)
 */
$cfg['Servers'][$i]['column_info']        = 'pma__column_info';
$cfg['Servers'][$i]['history']            = 'pma__history';
$cfg['Servers'][$i]['recent']             = 'pma__recent';
$cfg['Servers'][$i]['favorite']           = 'pma__favorite';
$cfg['Servers'][$i]['users']              = 'pma__users';
$cfg['Servers'][$i]['usergroups']         = 'pma__usergroups';

/**
 * Tables to avoid (potential memory issues)
 */
// $cfg['Servers'][$i]['tracking']         = 'pma__tracking';
// $cfg['Servers'][$i]['userconfig']       = 'pma__userconfig';
// $cfg['Servers'][$i]['designer_coords']  = 'pma__designer_coords';

/* ==================== PERFORMANCE & SECURITY ==================== */

/**
 * Memory limits for large databases
 */
$cfg['MemoryLimit'] = '512M';

/**
 * Security: zero configuration (disable auto-setup)
 */
$cfg['ZeroConf'] = false;

/**
 * Theme configuration
 */
$cfg['ThemeManager'] = true;
$cfg['ThemeDefault'] = 'pmahomme';

/**
 * Default language
 */
$cfg['DefaultLang'] = 'en';
$cfg['Lang'] = '';

/* ==================== FEATURE CONFIGURATION ==================== */

/**
 * Enable query history
 */
$cfg['QueryHistoryDB'] = true;
$cfg['QueryHistoryMax'] = 100;

/**
 * Navigation panel configuration
 */
$cfg['NavigationTreeEnableGrouping'] = true;
$cfg['NavigationTreeDbSeparator'] = '_';
$cfg['NavigationTreeTableSeparator'] = '__';

/**
 * End of configuration
 */
?>
```

## 4. MySQL SSL Configuration

### 4.1 Generate MySQL-Compatible SSL Certificates

#### 4.1.1 Certificate Generation Script for MySQL
Create `Generate-MySQL-SSL-Certs.ps1`:

```powershell
# MySQL SSL Certificate Generator
# Run as Administrator

param(
    [string]$OutputDir = "D:\laragon\data\mysql-9.1",
    [int]$KeySize = 2048,
    [int]$DaysValid = 3650
)

# Configuration
$country = "US"
$state = "State"
$locality = "City"
$organization = "Development"
$caCN = "MySQL Development CA"
$serverCN = "127.0.0.1"
$clientCN = "phpMyAdmin Client"

Write-Host "=== MySQL SSL Certificate Generation ===" -ForegroundColor Cyan
Write-Host "Output Directory: $OutputDir" -ForegroundColor Yellow

# Create directory if not exists
if (-not (Test-Path $OutputDir)) {
    New-Item -ItemType Directory -Path $OutputDir -Force | Out-Null
}

# Backup existing certificates
$backupDir = "$OutputDir\backup-$(Get-Date -Format 'yyyyMMdd-HHmmss')"
if (Test-Path "$OutputDir\*.pem") {
    New-Item -ItemType Directory -Path $backupDir -Force | Out-Null
    Move-Item "$OutputDir\*.pem" $backupDir -Force
    Write-Host "✓ Existing certificates backed up to: $backupDir" -ForegroundColor Green
}

# Subject information
$caSubject = "/C=$country/ST=$state/L=$locality/O=$organization/CN=$caCN"
$serverSubject = "/C=$country/ST=$state/L=$locality/O=$organization/CN=$serverCN"
$clientSubject = "/C=$country/ST=$state/L=$locality/O=$organization/CN=$clientCN"

Write-Host "`nGenerating Certificate Authority..." -ForegroundColor Cyan

# Step 1: Generate CA private key and certificate
openssl genrsa -out "$OutputDir\ca-key.pem" $KeySize
openssl req -new -x509 -nodes -days $DaysValid -key "$OutputDir\ca-key.pem" -out "$OutputDir\ca.pem" -subj $caSubject

Write-Host "✓ CA Certificate generated" -ForegroundColor Green

Write-Host "`nGenerating Server Certificate..." -ForegroundColor Cyan

# Step 2: Generate server certificate
openssl req -newkey rsa:$KeySize -nodes -keyout "$OutputDir\server-key.pem" -out "$OutputDir\server-req.pem" -subj $serverSubject
openssl rsa -in "$OutputDir\server-key.pem" -out "$OutputDir\server-key.pem"
openssl x509 -req -in "$OutputDir\server-req.pem" -days $DaysValid -CA "$OutputDir\ca.pem" -CAkey "$OutputDir\ca-key.pem" -set_serial 01 -out "$OutputDir\server-cert.pem"

Write-Host "✓ Server Certificate generated" -ForegroundColor Green

Write-Host "`nGenerating Client Certificate..." -ForegroundColor Cyan

# Step 3: Generate client certificate
openssl req -newkey rsa:$KeySize -nodes -keyout "$OutputDir\client-key.pem" -out "$OutputDir\client-req.pem" -subj $clientSubject
openssl rsa -in "$OutputDir\client-key.pem" -out "$OutputDir\client-key.pem"
openssl x509 -req -in "$OutputDir\client-req.pem" -days $DaysValid -CA "$OutputDir\ca.pem" -CAkey "$OutputDir\ca-key.pem" -set_serial 02 -out "$OutputDir\client-cert.pem"

Write-Host "✓ Client Certificate generated" -ForegroundColor Green

# Step 4: Verification
Write-Host "`nVerifying Certificates..." -ForegroundColor Cyan

$caVerify = openssl verify -CAfile "$OutputDir\ca.pem" "$OutputDir\ca.pem" 2>&1
$serverVerify = openssl verify -CAfile "$OutputDir\ca.pem" "$OutputDir\server-cert.pem" 2>&1
$clientVerify = openssl verify -CAfile "$OutputDir\ca.pem" "$OutputDir\client-cert.pem" 2>&1

if ($caVerify -match "OK") {
    Write-Host "✓ CA Certificate: Valid" -ForegroundColor Green
} else {
    Write-Host "✗ CA Certificate: $caVerify" -ForegroundColor Red
}

if ($serverVerify -match "OK") {
    Write-Host "✓ Server Certificate: Valid" -ForegroundColor Green
} else {
    Write-Host "✗ Server Certificate: $serverVerify" -ForegroundColor Red
}

if ($clientVerify -match "OK") {
    Write-Host "✓ Client Certificate: Valid" -ForegroundColor Green
} else {
    Write-Host "✗ Client Certificate: $clientVerify" -ForegroundColor Red
}

# Step 5: Set appropriate permissions (Windows)
Write-Host "`nSetting file permissions..." -ForegroundColor Cyan
Get-ChildItem "$OutputDir\*.pem" | ForEach-Object {
    $_.Attributes = "Normal"
}

Write-Host "`n=== Certificate Generation Complete ===" -ForegroundColor Green
Write-Host "Generated files:" -ForegroundColor Yellow
Get-ChildItem "$OutputDir\*.pem" | ForEach-Object {
    Write-Host "  • $($_.Name) ($([math]::Round($_.Length/1KB,2)) KB)" -ForegroundColor Gray
}

Write-Host "`nNext steps:" -ForegroundColor Cyan
Write-Host "1. Update MySQL configuration (my.ini)" -ForegroundColor Yellow
Write-Host "2. Restart MySQL service" -ForegroundColor Yellow
Write-Host "3. Update phpMyAdmin config.inc.php" -ForegroundColor Yellow
```

#### 4.1.2 Execute MySQL Certificate Generation
```powershell
# Run as Administrator
.\Generate-MySQL-SSL-Certs.ps1

# Optional: Specify custom directory
.\Generate-MySQL-SSL-Certs.ps1 -OutputDir "D:\laragon\etc\ssl\mysql" -DaysValid 365
```

### 4.2 Configure MySQL Server for SSL

Edit `D:\laragon\bin\mysql\mysql-9.1.0-winx64\my.ini`:

```ini
[client]
port=3306
socket=/tmp/mysql.sock

[mysqld]
# Basic Configuration
basedir="D:/laragon/bin/mysql/mysql-9.1.0-winx64"
datadir=D:/laragon/data/mysql-9.1
port=3306
socket=/tmp/mysql.sock

# Character Set
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci

# SSL Configuration (Essential)
ssl-ca = D:/laragon/data/mysql-9.1/ca.pem
ssl-cert = D:/laragon/data/mysql-9.1/server-cert.pem
ssl-key = D:/laragon/data/mysql-9.1/server-key.pem

# SSL Security Settings
ssl-cipher = TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256
tls_version = TLSv1.2,TLSv1.3

# Optional: Require SSL for all connections (development: OFF)
# require_secure_transport = ON

# Optional: SSL Certificate Verification
# ssl_ca = D:/laragon/data/mysql-9.1/ca.pem
# ssl_capath = 
# ssl_cert = D:/laragon/data/mysql-9.1/server-cert.pem
# ssl_cipher = 
# ssl_crl = 
# ssl_crlpath = 
# ssl_key = D:/laragon/data/mysql-9.1/server-key.pem

# Performance Settings
key_buffer_size = 256M
max_allowed_packet = 512M
table_open_cache = 256
sort_buffer_size = 1M
read_buffer_size = 1M
read_rnd_buffer_size = 4M
myisam_sort_buffer_size = 64M
thread_cache_size = 8
max_connections = 151

# Storage Engines
disabled_storage_engines="MyISAM,BLACKHOLE,FEDERATED,ARCHIVE,MRG_MYISAM"

# Security
secure-file-priv="D:/laragon/tmp"
explicit_defaults_for_timestamp=1

# Logging (Optional)
# general_log = 1
# general_log_file = D:/laragon/logs/mysql/general.log
# slow_query_log = 1
# slow_query_log_file = D:/laragon/logs/mysql/slow.log
# long_query_time = 2

[mysqldump]
quick
max_allowed_packet = 2G
```

### 4.3 Verify MySQL SSL Configuration

#### 4.3.1 Command Line Verification
```bash
# Connect to MySQL with SSL
mysql --ssl-ca="D:/laragon/data/mysql-9.1/ca.pem" ^
      --ssl-cert="D:/laragon/data/mysql-9.1/client-cert.pem" ^
      --ssl-key="D:/laragon/data/mysql-9.1/client-key.pem" ^
      -u root -p

# Check SSL status in MySQL
mysql> SHOW VARIABLES LIKE '%ssl%';
mysql> SHOW STATUS LIKE 'Ssl_cipher';
mysql> \s  -- View connection status
```

#### 4.3.2 PowerShell Verification Script
Create `Test-MySQL-SSL.ps1`:

```powershell
# Test MySQL SSL Connection
param(
    [string]$Username = "root",
    [string]$Password = "[YOUR_PASSWORD]",
    [string]$Hostname = "127.0.0.1",
    [int]$Port = 3306
)

$caCert = "D:/laragon/data/mysql-9.1/ca.pem"
$clientCert = "D:/laragon/data/mysql-9.1/client-cert.pem"
$clientKey = "D:/laragon/data/mysql-9.1/client-key.pem"

Write-Host "=== Testing MySQL SSL Connection ===" -ForegroundColor Cyan

# Test 1: Check certificate files exist
Write-Host "`n1. Certificate File Check:" -ForegroundColor Yellow
$files = @($caCert, $clientCert, $clientKey)
foreach ($file in $files) {
    if (Test-Path $file) {
        $size = [math]::Round((Get-Item $file).Length / 1KB, 2)
        Write-Host "  ✓ $(Split-Path $file -Leaf) ($size KB)" -ForegroundColor Green
    } else {
        Write-Host "  ✗ $(Split-Path $file -Leaf) (Missing)" -ForegroundColor Red
    }
}

# Test 2: Test SSL connection
Write-Host "`n2. SSL Connection Test:" -ForegroundColor Yellow

$connectionTest = @"
mysql --ssl-ca="$caCert" `
      --ssl-cert="$clientCert" `
      --ssl-key="$clientKey" `
      -h $Hostname `
      -P $Port `
      -u $Username `
      -p"$Password" `
      -e "STATUS" 2>&1
"@

try {
    $result = Invoke-Expression $connectionTest
    if ($result -match "SSL:\s+Cipher in use is") {
        $cipher = ($result | Select-String "SSL:\s+Cipher in use is (.+)").Matches.Groups[1].Value
        Write-Host "  ✓ SSL Connected successfully" -ForegroundColor Green
        Write-Host "    Cipher: $cipher" -ForegroundColor Gray
    } else {
        Write-Host "  ✗ SSL Connection failed" -ForegroundColor Red
        Write-Host "    Output: $result" -ForegroundColor Gray
    }
} catch {
    Write-Host "  ✗ Connection test error: $_" -ForegroundColor Red
}

# Test 3: Test without SSL (should fail if require_secure_transport=ON)
Write-Host "`n3. Non-SSL Connection Test:" -ForegroundColor Yellow

$nonSslTest = @"
mysql --skip-ssl `
      -h $Hostname `
      -P $Port `
      -u $Username `
      -p"$Password" `
      -e "SELECT 'Non-SSL test'" 2>&1
"@

try {
    $result = Invoke-Expression $nonSslTest
    if ($result -match "Non-SSL test") {
        Write-Host "  ⚠ Non-SSL connection allowed" -ForegroundColor Yellow
    } else {
        Write-Host "  ✓ Non-SSL connection blocked (expected)" -ForegroundColor Green
    }
} catch {
    Write-Host "  ✓ Non-SSL connection failed (expected)" -ForegroundColor Green
}

Write-Host "`n=== Test Complete ===" -ForegroundColor Cyan
```

## 5. Configuration Storage Setup

### 5.1 Create phpMyAdmin Configuration Database

#### 5.1.1 SQL Script for Configuration Storage
Create `setup-pma-storage.sql`:

```sql
-- phpMyAdmin SQL Dump for Configuration Storage
-- Version: 6.0
-- Generated: December 2025

SET SQL_MODE = "NO_AUTO_VALUE_ON_ZERO";
START TRANSACTION;
SET time_zone = "+00:00";

-- --------------------------------------------------------
-- Create database if not exists
-- --------------------------------------------------------
CREATE DATABASE IF NOT EXISTS `phpmyadmin` 
  CHARACTER SET utf8 COLLATE utf8_bin;
USE `phpmyadmin`;

-- --------------------------------------------------------
-- User creation and privileges
-- --------------------------------------------------------
CREATE USER IF NOT EXISTS 'pma'@'localhost' 
  IDENTIFIED BY [YOUR_PASSWORD]';

GRANT ALL PRIVILEGES ON `phpmyadmin`.* TO 'pma'@'localhost';
GRANT SELECT ON `mysql`.* TO 'pma'@'localhost';

FLUSH PRIVILEGES;

-- --------------------------------------------------------
-- Table structure for table `pma__bookmark`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__bookmark` (
  `id` int(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `dbase` varchar(255) COLLATE utf8_bin NOT NULL DEFAULT '',
  `user` varchar(255) COLLATE utf8_bin NOT NULL DEFAULT '',
  `label` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL DEFAULT '',
  `query` text COLLATE utf8_bin NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Bookmarks';

-- --------------------------------------------------------
-- Table structure for table `pma__relation`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__relation` (
  `master_db` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `master_table` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `master_field` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `foreign_db` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `foreign_table` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `foreign_field` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  PRIMARY KEY (`master_db`,`master_table`,`master_field`),
  KEY `foreign_field` (`foreign_db`,`foreign_table`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Relation table';

-- --------------------------------------------------------
-- Table structure for table `pma__table_info`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__table_info` (
  `db_name` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `table_name` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `display_field` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  PRIMARY KEY (`db_name`,`table_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Table information for phpMyAdmin';

-- --------------------------------------------------------
-- Table structure for table `pma__table_coords`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__table_coords` (
  `db_name` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `table_name` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `pdf_page_number` int(11) NOT NULL DEFAULT 0,
  `x` float UNSIGNED NOT NULL DEFAULT 0,
  `y` float UNSIGNED NOT NULL DEFAULT 0,
  PRIMARY KEY (`db_name`,`table_name`,`pdf_page_number`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Table coordinates for phpMyAdmin PDF output';

-- --------------------------------------------------------
-- Table structure for table `pma__pdf_pages`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__pdf_pages` (
  `db_name` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `page_nr` int(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `page_descr` varchar(50) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL DEFAULT '',
  PRIMARY KEY (`page_nr`),
  KEY `db_name` (`db_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='PDF relation pages for phpMyAdmin';

-- --------------------------------------------------------
-- Table structure for table `pma__column_info`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__column_info` (
  `id` int(5) UNSIGNED NOT NULL AUTO_INCREMENT,
  `db_name` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `table_name` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `column_name` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `comment` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL DEFAULT '',
  `mimetype` varchar(255) CHARACTER SET utf8 COLLATE utf8_general_ci NOT NULL DEFAULT '',
  `transformation` varchar(255) COLLATE utf8_bin NOT NULL DEFAULT '',
  `transformation_options` varchar(255) COLLATE utf8_bin NOT NULL DEFAULT '',
  `input_transformation` varchar(255) COLLATE utf8_bin NOT NULL DEFAULT '',
  `input_transformation_options` varchar(255) COLLATE utf8_bin NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  UNIQUE KEY `db_name` (`db_name`,`table_name`,`column_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Column information for phpMyAdmin';

-- --------------------------------------------------------
-- Table structure for table `pma__history`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__history` (
  `id` bigint(20) UNSIGNED NOT NULL AUTO_INCREMENT,
  `username` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `db` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `table` varchar(64) COLLATE utf8_bin NOT NULL DEFAULT '',
  `timevalue` timestamp NOT NULL DEFAULT current_timestamp(),
  `sqlquery` text COLLATE utf8_bin NOT NULL,
  PRIMARY KEY (`id`),
  KEY `username` (`username`,`db`,`table`,`timevalue`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='SQL history for phpMyAdmin';

-- --------------------------------------------------------
-- Table structure for table `pma__recent`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__recent` (
  `username` varchar(64) COLLATE utf8_bin NOT NULL,
  `tables` text COLLATE utf8_bin NOT NULL,
  PRIMARY KEY (`username`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Recently accessed tables';

-- --------------------------------------------------------
-- Table structure for table `pma__favorite`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__favorite` (
  `username` varchar(64) COLLATE utf8_bin NOT NULL,
  `tables` text COLLATE utf8_bin NOT NULL,
  PRIMARY KEY (`username`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Favorite tables';

-- --------------------------------------------------------
-- Table structure for table `pma__users`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__users` (
  `username` varchar(64) COLLATE utf8_bin NOT NULL,
  `usergroup` varchar(64) COLLATE utf8_bin NOT NULL,
  PRIMARY KEY (`username`,`usergroup`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Users and their assignments to user groups';

-- --------------------------------------------------------
-- Table structure for table `pma__usergroups`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__usergroups` (
  `usergroup` varchar(64) COLLATE utf8_bin NOT NULL,
  `tab` varchar(64) COLLATE utf8_bin NOT NULL,
  `allowed` enum('Y','N') COLLATE utf8_bin NOT NULL DEFAULT 'N',
  PRIMARY KEY (`usergroup`,`tab`,`allowed`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='User groups with configured menu items';

-- --------------------------------------------------------
-- Table structure for table `pma__navigationhiding`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__navigationhiding` (
  `username` varchar(64) COLLATE utf8_bin NOT NULL,
  `item_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `item_type` varchar(64) COLLATE utf8_bin NOT NULL,
  `db_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `table_name` varchar(64) COLLATE utf8_bin NOT NULL,
  PRIMARY KEY (`username`,`item_name`,`item_type`,`db_name`,`table_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Hidden items of navigation tree';

-- --------------------------------------------------------
-- Table structure for table `pma__savedsearches`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__savedsearches` (
  `id` int(5) UNSIGNED NOT NULL AUTO_INCREMENT,
  `username` varchar(64) COLLATE utf8_bin NOT NULL,
  `db_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `search_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `search_data` text COLLATE utf8_bin NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `u_savedsearches_username_dbname` (`username`,`db_name`,`search_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Saved searches';

-- --------------------------------------------------------
-- Table structure for table `pma__central_columns`
-- --------------------------------------------------------
CREATE TABLE IF NOT EXISTS `pma__central_columns` (
  `db_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `col_name` varchar(64) COLLATE utf8_bin NOT NULL,
  `col_type` varchar(64) COLLATE utf8_bin NOT NULL,
  `col_length` text COLLATE utf8_bin DEFAULT NULL,
  `col_collation` varchar(64) COLLATE utf8_bin NOT NULL,
  `col_isNull` tinyint(1) NOT NULL,
  `col_extra` varchar(255) COLLATE utf8_bin DEFAULT '',
  `col_default` text COLLATE utf8_bin DEFAULT NULL,
  PRIMARY KEY (`db_name`,`col_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin COMMENT='Central list of columns';

COMMIT;
```

#### 5.1.2 Import Configuration Storage
```bash
# Method 1: Using mysql command line
mysql -u root -p < setup-pma-storage.sql

# Method 2: Using phpMyAdmin interface
# 1. Login to phpMyAdmin
# 2. Click the warning message about configuration storage
# 3. Follow the wizard to create database and tables
```

## 6. Troubleshooting Guide

### 6.1 Common Issues and Solutions

#### Issue 1: "Cannot connect to MySQL using SSL"
**Symptoms**: Connection errors with SSL-related messages
**Solutions**:
1. Verify certificate paths in `config.inc.php` and `my.ini`
2. Check file permissions: Certificates should be readable by MySQL service
3. Ensure certificates are not corrupted:
   ```powershell
   openssl verify -CAfile ca.pem server-cert.pem
   ```

#### Issue 2: Memory Exhaustion Errors
**Symptoms**: "Allowed memory size exhausted" in phpMyAdmin
**Solutions**:
1. Increase PHP memory limit:
   ```ini
   ; In php.ini
   memory_limit = 512M
   ```
2. Disable non-essential configuration storage tables
3. Clear phpMyAdmin cache in `D:\laragon\tmp\`

#### Issue 3: Certificate Verification Warnings
**Symptoms**: "SSL is used with disabled verification"
**Solutions**:
1. For development: Accept this warning (secure enough)
2. For production-like setup: Generate proper certificates with correct CN
3. Set `ssl_verify = true` only after certificate validation passes

#### Issue 4: Mixed Content Warnings
**Symptoms**: Browser shows "Not Secure" despite HTTPS
**Solutions**:
1. Ensure `$cfg['ForceSSL'] = true` is set
2. Check Apache SSL configuration is active
3. Clear browser cache and hard reload (Ctrl+F5)

### 6.2 Diagnostic Tools

#### 6.2.1 SSL Configuration Check Script
Create `diagnose-ssl.ps1`:

```powershell
# Comprehensive SSL Diagnostics Script
Write-Host "=== PHPMyAdmin SSL Diagnostics ===" -ForegroundColor Cyan
Write-Host "Timestamp: $(Get-Date)" -ForegroundColor Gray

# Check Apache SSL
Write-Host "`n1. Apache SSL Configuration:" -ForegroundColor Yellow
$apacheConf = "D:\laragon\etc\apache2\httpd.conf"
if (Test-Path $apacheConf) {
    $sslModule = Select-String -Path $apacheConf -Pattern "mod_ssl"
    if ($sslModule) {
        Write-Host "  ✓ mod_ssl is loaded" -ForegroundColor Green
    } else {
        Write-Host "  ✗ mod_ssl not found in httpd.conf" -ForegroundColor Red
    }
}

# Check phpMyAdmin SSL Settings
Write-Host "`n2. phpMyAdmin Configuration:" -ForegroundColor Yellow
$pmaConfig = "D:\laragon\etc\apps\phpmyadmin6\config.inc.php"
if (Test-Path $pmaConfig) {
    $content = Get-Content $pmaConfig -Raw
    $forceSSL = $content -match "ForceSSL.*=.*true"
    $sslEnabled = $content -match "ssl.*=.*true"
    
    Write-Host "  ForceSSL: $(if($forceSSL){'✓ Enabled'}else{'✗ Disabled'})" -ForegroundColor $(if($forceSSL){'Green'}else{'Red'})
    Write-Host "  MySQL SSL: $(if($sslEnabled){'✓ Enabled'}else{'✗ Disabled'})" -ForegroundColor $(if($sslEnabled){'Green'}else{'Red'})
}

# Check Certificate Files
Write-Host "`n3. Certificate Files:" -ForegroundColor Yellow
$certPaths = @(
    "D:\laragon\etc\apps\ssl\localhost.crt",
    "D:\laragon\etc\apps\ssl\localhost.key",
    "D:\laragon\data\mysql-9.1\ca.pem",
    "D:\laragon\data\mysql-9.1\server-cert.pem",
    "D:\laragon\data\mysql-9.1\client-cert.pem"
)

foreach ($cert in $certPaths) {
    if (Test-Path $cert) {
        $size = [math]::Round((Get-Item $cert).Length / 1KB, 2)
        $valid = $true
        
        if ($cert -match "\.crt$|\.pem$") {
            $certCheck = openssl x509 -in $cert -text -noout 2>&1
            if ($certCheck -match "error") {
                $valid = $false
            }
        }
        
        Write-Host "  $(if($valid){'✓'}else{'✗'}) $(Split-Path $cert -Leaf) ($size KB)" -ForegroundColor $(if($valid){'Green'}else{'Red'})
    } else {
        Write-Host "  ✗ $(Split-Path $cert -Leaf) (Missing)" -ForegroundColor Red
    }
}

# Test MySQL SSL Connection
Write-Host "`n4. MySQL SSL Test:" -ForegroundColor Yellow
$mysqlTest = @{
    'SSL Enabled' = "SHOW VARIABLES LIKE 'have_ssl'"
    'SSL Cipher' = "SHOW STATUS LIKE 'Ssl_cipher'"
    'SSL Version' = "SHOW STATUS LIKE 'Ssl_version'"
}

foreach ($test in $mysqlTest.GetEnumerator()) {
    $result = mysql -u root -p"[YOUR_PASSWORD]" -e "$($test.Value)" 2>&1 | Select-String "Value"
    if ($result) {
        $value = ($result -split "\s+")[-1]
        Write-Host "  $($test.Key): $value" -ForegroundColor Green
    } else {
        Write-Host "  $($test.Key): Not available" -ForegroundColor Red
    }
}

Write-Host "`n=== Diagnostics Complete ===" -ForegroundColor Cyan
```

## 7. Maintenance Procedures

### 7.1 Certificate Renewal Schedule
- **Development certificates**: Renew every 1-2 years
- **Production-like certificates**: Follow organizational PKI policies
- **MySQL auto-generated certificates**: MySQL 9.1+ auto-renews on startup

### 7.2 Backup Procedures
```powershell
# Backup SSL Certificates
$backupDir = "D:\laragon-backup\ssl-$(Get-Date -Format 'yyyyMMdd')"
New-Item -ItemType Directory -Path $backupDir -Force

# Copy Apache certificates
Copy-Item "D:\laragon\etc\apps\ssl\*" $backupDir\

# Copy MySQL certificates  
Copy-Item "D:\laragon\data\mysql-9.1\*.pem" $backupDir\

# Backup configurations
Copy-Item "D:\laragon\etc\apps\phpmyadmin6\config.inc.php" "$backupDir\config.inc.php.backup"
Copy-Item "D:\laragon\bin\mysql\mysql-9.1.0-winx64\my.ini" "$backupDir\my.ini.backup"

Write-Host "Backup completed: $backupDir" -ForegroundColor Green
```

### 7.3 Monitoring and Logging
- Enable Apache SSL logging in virtual host configuration
- Monitor MySQL error log for SSL-related issues
- Regular review of phpMyAdmin access logs

## 8. Security Best Practices

### 8.1 Development Environment
1. **Use self-signed certificates** with proper CN/SAN
2. **Keep `ssl_verify = false`** for local development
3. **Regularly update** Laragon components
4. **Use strong passwords** for MySQL and phpMyAdmin

### 8.2 Production Considerations
1. **Obtain certificates from trusted CA**
2. **Enable full certificate verification** (`ssl_verify = true`)
3. **Implement proper firewall rules**
4. **Regular security audits** of SSL/TLS configuration

## 9. Appendices

### Appendix A: Quick Reference Commands

```bash
# Restart services
laragon restart all

# Generate certificates
.\Generate-LocalhostCert.ps1
.\Generate-MySQL-SSL-Certs.ps1

# Test connections
.\Test-MySQL-SSL.ps1
.\diagnose-ssl.ps1

# MySQL SSL commands
mysql --ssl-ca=ca.pem --ssl-cert=client-cert.pem --ssl-key=client-key.pem -u root -p
SHOW VARIABLES LIKE '%ssl%';
```

### Appendix B: Configuration File Locations
- **Apache**: `D:\laragon\etc\apache2\`
- **phpMyAdmin**: `D:\laragon\etc\apps\phpmyadmin6\`
- **MySQL**: `D:\laragon\bin\mysql\mysql-9.1.0-winx64\`
- **SSL Certificates**: `D:\laragon\etc\apps\ssl\` and `D:\laragon\data\mysql-9.1\`

### Appendix C: Useful Resources
- [phpMyAdmin Documentation](https://docs.phpmyadmin.net/)
- [MySQL SSL Configuration Guide](https://dev.mysql.com/doc/refman/9.1/en/using-encrypted-connections.html)
- [OpenSSL Command Reference](https://www.openssl.org/docs/)

---

## Acknowledgements

### Contributors
- **Author**: System Architect - Initial documentation and script development
- **Reviewers**: Security Officer, IT Manager - Technical review and approval
- **Community**: Thanks to the open-source community for feedback and improvements

### Technologies & Tools
- **Laragon**: Excellent local development environment
- **phpMyAdmin**: Powerful MySQL administration tool
- **MySQL**: Robust database server with SSL support
- **OpenSSL**: Industry-standard cryptography library
- **PowerShell**: Powerful scripting for Windows automation

### Resources & Inspiration
- [phpMyAdmin Documentation](https://docs.phpmyadmin.net/)
- [MySQL SSL Configuration Guide](https://dev.mysql.com/doc/refman/9.1/en/using-encrypted-connections.html)
- [OpenSSL Documentation](https://www.openssl.org/docs/)
- [Laragon Community](https://forum.laragon.org/)
- Various security best practices from OWASP and NIST

### Special Thanks
- The developers of Laragon for creating such a user-friendly development environment
- The phpMyAdmin team for their comprehensive database management tool
- The MySQL team for robust SSL/TLS implementation
- All contributors who help improve this documentation

---

## Document Approval

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Author | System Architect | | |
| Reviewer | Security Officer | | |
| Approver | IT Manager | | |

**Document Version History**
- v1.0 (Dec 2025): Initial release with comprehensive SSL guidance

---
*This document provides guidance for SSL configuration in development environments. For production deployments, consult with security professionals and follow organizational security policies.*