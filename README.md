# Varnish Cache Installation Guide

Varnish Cache is a high-performance HTTP accelerator designed for content-heavy dynamic websites as well as APIs. It sits between your web server and clients, caching frequently requested content to dramatically improve response times and reduce server load.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Supported Operating Systems](#supported-operating-systems)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Service Management](#service-management)
6. [Troubleshooting](#troubleshooting)
7. [Security Considerations](#security-considerations)
8. [Performance Tuning](#performance-tuning)
9. [Backup and Restore](#backup-and-restore)
10. [System Requirements](#system-requirements)
11. [Support](#support)
12. [Contributing](#contributing)
13. [License](#license)
14. [Acknowledgments](#acknowledgments)
15. [Version History](#version-history)
16. [Appendices](#appendices)

## 1. Prerequisites

- Operating system: Linux, macOS, or Windows (WSL)
- Minimum 1GB RAM (more for larger caches)
- Backend web server (Apache, nginx, etc.)
- Administrative/sudo access
- Port 80 available (or custom port)


## 2. Supported Operating Systems

This guide supports installation on:
- RHEL 8/9 and derivatives (CentOS Stream, Rocky Linux, AlmaLinux)
- Debian 11/12
- Ubuntu 20.04/22.04/24.04 LTS
- Arch Linux (rolling release)
- Alpine Linux 3.18+
- openSUSE Leap 15.5+ / Tumbleweed
- SUSE Linux Enterprise Server (SLES) 15+
- macOS 12+ (Monterey and later) 
- FreeBSD 13+
- Windows 10/11/Server 2019+ (where applicable)

## 3. Installation

### RHEL/CentOS/Rocky Linux/AlmaLinux

```bash
# RHEL/CentOS 7
sudo yum install -y epel-release
sudo yum install -y varnish

# RHEL/CentOS/Rocky/AlmaLinux 8+
sudo dnf install -y epel-release
sudo dnf install -y varnish

# Varnish official repository (latest version)
# Add repository
curl -s https://packagecloud.io/install/repositories/varnishcache/varnish73/script.rpm.sh | sudo bash

# Install Varnish
sudo yum install -y varnish

# Enable and start service
sudo systemctl enable --now varnish
```

### Debian/Ubuntu

```bash
# Distribution packages
sudo apt update
sudo apt install -y varnish

# Varnish official repository (latest version)
# Install dependencies
sudo apt install -y debian-archive-keyring curl gnupg apt-transport-https

# Add repository
curl -fsSL https://packagecloud.io/varnishcache/varnish73/gpgkey | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/varnish.gpg
echo "deb https://packagecloud.io/varnishcache/varnish73/ubuntu/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/varnish.list

# Install Varnish
sudo apt update
sudo apt install -y varnish

# Enable and start service
sudo systemctl enable --now varnish
```

### Alpine Linux

```bash
# Install Varnish
apk add --no-cache varnish

# Install additional tools
apk add --no-cache varnish-libs varnish-dev

# Enable and start service
rc-update add varnishd default
rc-service varnishd start
```

### macOS

```bash
# Using Homebrew
brew install varnish

# Start Varnish
brew services start varnish

# Or run manually
varnishd -F -a :8080 -f /usr/local/etc/varnish/default.vcl
```

### FreeBSD

```bash
# Using pkg
pkg install varnish7

# Enable in rc.conf
echo 'varnishd_enable="YES"' >> /etc/rc.conf

# Start service
service varnishd start
```

### Build from Source

```bash
# Install dependencies
# RHEL/CentOS
sudo yum groupinstall -y "Development Tools"
sudo yum install -y python3 python3-sphinx libedit-devel pcre-devel ncurses-devel

# Debian/Ubuntu
sudo apt install -y build-essential python3 python3-sphinx libedit-dev libpcre3-dev libncurses-dev

# Alpine
apk add --no-cache build-base python3 py3-sphinx libedit-dev pcre-dev ncurses-dev

# Download and build
wget https://varnish-cache.org/_downloads/varnish-7.3.0.tgz
tar -zxvf varnish-7.3.0.tgz
cd varnish-7.3.0

./configure --prefix=/usr/local
make
sudo make install
```

## 4. Configuration

### Basic Configuration File

Create `/etc/varnish/default.vcl`:
```vcl
vcl 4.1;

import std;

# Default backend definition
backend default {
    .host = "127.0.0.1";
    .port = "8080";
    .probe = {
        .url = "/";
        .interval = 5s;
        .timeout = 1s;
        .window = 5;
        .threshold = 3;
    }
}

# Access control list for purging
acl purge {
    "localhost";
    "127.0.0.1";
    "::1";
}

# Client side
sub vcl_recv {
    # Normalize the header, remove the port
    set req.http.Host = regsub(req.http.Host, ":[0-9]+", "");
    
    # Normalize the query arguments
    set req.url = std.querysort(req.url);
    
    # Strip hash, server doesn't need it
    if (req.url ~ "\#") {
        set req.url = regsub(req.url, "\#.*$", "");
    }
    
    # Strip trailing ?
    if (req.url ~ "\?$") {
        set req.url = regsub(req.url, "\?$", "");
    }
    
    # Handle PURGE requests
    if (req.method == "PURGE") {
        if (client.ip !~ purge) {
            return (synth(405, "Not allowed."));
        }
        return (purge);
    }
    
    # Only cache GET or HEAD requests
    if (req.method != "GET" && req.method != "HEAD") {
        return (pass);
    }
    
    # Don't cache if cookies are present
    if (req.http.Authorization || req.http.Cookie) {
        return (pass);
    }
    
    # Cache static files
    if (req.url ~ "\.(jpg|jpeg|gif|png|ico|css|zip|tgz|gz|rar|bz2|pdf|txt|tar|wav|bmp|rtf|js|flv|swf|html|htm)$") {
        unset req.http.Cookie;
        return (hash);
    }
    
    return (hash);
}

sub vcl_backend_response {
    # Set default TTL
    if (!beresp.http.Cache-Control) {
        set beresp.ttl = 1h;
    }
    
    # Cache static files for longer
    if (bereq.url ~ "\.(jpg|jpeg|gif|png|ico|css|zip|tgz|gz|rar|bz2|pdf|txt|tar|wav|bmp|rtf|js|flv|swf|html|htm)$") {
        unset beresp.http.Set-Cookie;
        set beresp.ttl = 7d;
    }
    
    # Enable grace mode
    set beresp.grace = 1h;
    
    return (deliver);
}

sub vcl_deliver {
    # Add debug headers
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
        set resp.http.X-Cache-Hits = obj.hits;
    } else {
        set resp.http.X-Cache = "MISS";
    }
    
    # Remove some headers
    unset resp.http.X-Powered-By;
    unset resp.http.Server;
    unset resp.http.Via;
    unset resp.http.X-Varnish;
    
    return (deliver);
}
```

### Service Configuration

#### RHEL/CentOS/Debian/Ubuntu (systemd)

Edit `/etc/systemd/system/varnish.service.d/override.conf`:
```ini
[Service]
ExecStart=
ExecStart=/usr/sbin/varnishd \
    -a :80 \
    -T localhost:6082 \
    -f /etc/varnish/default.vcl \
    -s malloc,256m \
    -p default_ttl=120 \
    -p default_grace=3600
```

Apply changes:
```bash
sudo systemctl daemon-reload
sudo systemctl restart varnish
```

#### Alpine Linux

Edit `/etc/conf.d/varnishd`:
```bash
# Varnish configuration
VARNISHD_OPTS="-a :80 \
    -T localhost:6082 \
    -f /etc/varnish/default.vcl \
    -s malloc,256m \
    -p default_ttl=120 \
    -p default_grace=3600"
```

### Storage Backends

```bash
# Memory storage (fast, limited by RAM)
-s malloc,1g

# File storage (slower, more capacity)
-s file,/var/lib/varnish/varnish_storage.bin,10g

# Multiple storage backends
-s memory=malloc,256m \
-s disk=file,/var/lib/varnish/varnish_storage.bin,10g
```

## Advanced VCL Examples

### WordPress Configuration

```vcl
vcl 4.1;

import std;

backend default {
    .host = "127.0.0.1";
    .port = "8080";
}

sub vcl_recv {
    # Don't cache WordPress admin
    if (req.url ~ "wp-(admin|login|cron)" || req.url ~ "preview=true") {
        return (pass);
    }
    
    # Don't cache logged-in users
    if (req.http.Cookie ~ "wordpress_logged_in_") {
        return (pass);
    }
    
    # Remove marketing cookies
    if (req.http.Cookie) {
        set req.http.Cookie = regsuball(req.http.Cookie, "__utm.=[^;]+(; )?", "");
        set req.http.Cookie = regsuball(req.http.Cookie, "_ga=[^;]+(; )?", "");
        set req.http.Cookie = regsuball(req.http.Cookie, "_gid=[^;]+(; )?", "");
        set req.http.Cookie = regsuball(req.http.Cookie, "_gat=[^;]+(; )?", "");
        set req.http.Cookie = regsuball(req.http.Cookie, "utmctr=[^;]+(; )?", "");
        set req.http.Cookie = regsuball(req.http.Cookie, "utmcmd.=[^;]+(; )?", "");
        set req.http.Cookie = regsuball(req.http.Cookie, "utmccn.=[^;]+(; )?", "");
        
        # Remove empty cookies
        if (req.http.Cookie == "") {
            unset req.http.Cookie;
        }
    }
    
    return (hash);
}

sub vcl_backend_response {
    # Cache everything for 1 hour by default
    set beresp.ttl = 1h;
    
    # Don't cache error pages
    if (beresp.status >= 400) {
        set beresp.ttl = 0s;
    }
    
    # Allow stale content for 24 hours
    set beresp.grace = 24h;
    
    return (deliver);
}
```

### Load Balancing

```vcl
vcl 4.1;

import directors;
import std;

# Define backends
backend web1 {
    .host = "192.168.1.10";
    .port = "80";
    .probe = {
        .url = "/health";
        .interval = 5s;
        .timeout = 1s;
        .window = 5;
        .threshold = 3;
    }
}

backend web2 {
    .host = "192.168.1.11";
    .port = "80";
    .probe = {
        .url = "/health";
        .interval = 5s;
        .timeout = 1s;
        .window = 5;
        .threshold = 3;
    }
}

# Initialize director
sub vcl_init {
    new web_director = directors.round_robin();
    web_director.add_backend(web1);
    web_director.add_backend(web2);
}

sub vcl_recv {
    set req.backend_hint = web_director.backend();
    return (hash);
}
```

### ESI (Edge Side Includes)

```vcl
vcl 4.1;

backend default {
    .host = "127.0.0.1";
    .port = "8080";
}

sub vcl_recv {
    # Enable ESI processing
    set req.http.Surrogate-Capability = "varnish=ESI/1.0";
    return (hash);
}

sub vcl_backend_response {
    # Check for ESI
    if (beresp.http.Surrogate-Control ~ "ESI/1.0") {
        unset beresp.http.Surrogate-Control;
        set beresp.do_esi = true;
        set beresp.ttl = 24h;
    }
    
    return (deliver);
}
```

## 8. Performance Tuning

### System Parameters

Add to `/etc/sysctl.conf`:
```bash
# Network tuning
net.core.somaxconn = 1024
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.tcp_max_syn_backlog = 4096
net.ipv4.tcp_tw_reuse = 1

# Memory tuning
vm.swappiness = 10

# Apply changes
sudo sysctl -p
```

### Varnish Parameters

```bash
# Thread pools (2 per CPU core)
varnishd -p thread_pools=4

# Thread pool size
varnishd -p thread_pool_min=100 -p thread_pool_max=4000

# Workspace size
varnishd -p workspace_backend=256k -p workspace_client=256k

# Listen queue depth
varnishd -p listen_depth=4096
```

### Storage Tuning

```bash
# For file storage, use separate disk
sudo mkdir -p /var/lib/varnish
sudo mount /dev/sdb1 /var/lib/varnish

# Use memory-mapped file
varnishd -s file,/var/lib/varnish/varnish_storage.bin,100g,1g

# Parameters: filename,size,granularity
```

## Monitoring and Statistics

### varnishstat

```bash
# Real-time statistics
varnishstat

# One-time output
varnishstat -1

# Specific counters
varnishstat -f MAIN.cache_hit -f MAIN.cache_miss

# JSON output
varnishstat -j
```

### varnishlog

```bash
# Live log
varnishlog

# Client requests
varnishlog -g request

# Backend requests
varnishlog -g backend

# Filter by URL
varnishlog -q 'ReqURL ~ "^/api/"'

# Filter by response code
varnishlog -q 'RespStatus == 404'
```

### varnishtop

```bash
# Most requested URLs
varnishtop -i requrl

# Most common user agents
varnishtop -i reqheader -I User-Agent

# Backend requests
varnishtop -b -i requrl
```

### varnishhist

```bash
# Response time histogram
varnishhist

# Client requests only
varnishhist -c

# Backend requests only
varnishhist -b
```

## Security Configuration

### ACL for Administration

```vcl
acl admin {
    "localhost";
    "127.0.0.1";
    "::1";
    "192.168.1.0"/24;
}

sub vcl_recv {
    # Restrict purge
    if (req.method == "PURGE") {
        if (client.ip !~ admin) {
            return (synth(403, "Forbidden"));
        }
        return (purge);
    }
    
    # Restrict ban
    if (req.method == "BAN") {
        if (client.ip !~ admin) {
            return (synth(403, "Forbidden"));
        }
        ban("req.http.host == " + req.http.host);
        return (synth(200, "Banned"));
    }
}
```

### Rate Limiting

```vcl
import vsthrottle;

sub vcl_recv {
    # Rate limit: 100 requests per 60 seconds per IP
    if (vsthrottle.is_denied(client.ip, 100, 60s)) {
        return (synth(429, "Too Many Requests"));
    }
}
```

### Security Headers

```vcl
sub vcl_deliver {
    # Security headers
    set resp.http.X-Frame-Options = "SAMEORIGIN";
    set resp.http.X-Content-Type-Options = "nosniff";
    set resp.http.X-XSS-Protection = "1; mode=block";
    set resp.http.Referrer-Policy = "strict-origin-when-cross-origin";
    
    # HSTS (only with HTTPS)
    if (req.http.X-Forwarded-Proto == "https") {
        set resp.http.Strict-Transport-Security = "max-age=31536000; includeSubDomains";
    }
}
```

## Integration with Web Servers

### nginx Backend

```nginx
server {
    listen 8080;
    server_name example.com;
    
    # Real IP from Varnish
    set_real_ip_from 127.0.0.1;
    real_ip_header X-Forwarded-For;
    
    location / {
        root /var/www/html;
        index index.html;
    }
    
    # Health check endpoint
    location /health {
        access_log off;
        return 200 "OK";
    }
}
```

### Apache Backend

```apache
Listen 8080
<VirtualHost *:8080>
    ServerName example.com
    DocumentRoot /var/www/html
    
    # Real IP from Varnish
    RemoteIPHeader X-Forwarded-For
    RemoteIPInternalProxy 127.0.0.1
    
    # Logging with real IP
    LogFormat "%a %l %u %t \"%r\" %>s %b" varnish
    CustomLog /var/log/apache2/access.log varnish
</VirtualHost>
```

## 6. Troubleshooting

### Common Issues

1. **503 Backend fetch failed**:
```bash
# Check backend health
varnishadm backend.list

# Test backend directly
curl -I http://127.0.0.1:8080/

# Check VCL syntax
varnishd -C -f /etc/varnish/default.vcl
```

2. **Low hit rate**:
```bash
# Check what's not being cached
varnishlog -g request -q "VCL_call eq 'PASS'"

# Check cookies
varnishlog -g request -i ReqHeader -I Cookie
```

3. **Memory issues**:
```bash
# Check memory usage
varnishstat -f MAIN.g_bytes

# Adjust storage size
sudo systemctl edit varnish
# Change -s malloc,256m to appropriate size
```

### Debug Mode

```vcl
sub vcl_deliver {
    # Debug headers (remove in production)
    set resp.http.X-Cache-Hits = obj.hits;
    set resp.http.X-Cache-TTL = obj.ttl;
    set resp.http.X-Cache-Grace = obj.grace;
    
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
    } else {
        set resp.http.X-Cache = "MISS";
    }
}
```

## Maintenance

### Cache Invalidation

```bash
# Purge specific URL
curl -X PURGE http://localhost/path/to/page

# Ban pattern
varnishadm ban "req.url ~ ^/images/"

# Clear entire cache
varnishadm ban "req.url ~ ."
```

### 9. Backup and Restore

```bash
# Backup VCL
cp /etc/varnish/default.vcl /backup/default.vcl.$(date +%Y%m%d)

# Test new VCL
varnishd -C -f /etc/varnish/new.vcl

# Load new VCL without restart
varnishadm vcl.load newconfig /etc/varnish/new.vcl
varnishadm vcl.use newconfig
```

### Updates

```bash
# RHEL/CentOS
sudo yum update varnish

# Debian/Ubuntu
sudo apt update && sudo apt upgrade varnish

# Check version
varnishd -V
```

## Additional Resources

- [Official Documentation](https://varnish-cache.org/docs/)
- [VCL Reference](https://varnish-cache.org/docs/trunk/reference/vcl.html)
- [Varnish Book](https://book.varnish-software.com/)
- [GitHub Repository](https://github.com/varnishcache/varnish-cache)
- [Community Forum](https://varnish-cache.org/lists/)
- [Varnish Modules (VMODs)](https://varnish-cache.org/vmods/)

---

**Note:** This guide is part of the [HowToMgr](https://howtomgr.github.io) collection. Always refer to official documentation for the most up-to-date information.