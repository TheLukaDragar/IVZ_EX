# FreeRADIUS Server Setup and Configuration Guide

## Introduction

This guide covers the setup and configuration of a FreeRADIUS server instance, including authentication setup, HTTP integration, and roaming capabilities.

## Initial Setup

### Software Installation

Install the required packages:

```bash
sudo apt update
sudo apt install freeradius freeradius-utils apache2 libapache2-mod-auth-radius wireshark
```

During Wireshark installation, select "yes" when asked "Should non-superusers be able to capture packets?". If needed, reconfigure using:

```bash
sudo dpkg-reconfigure wireshark-common
```

Add your user to the wireshark group:

```bash
sudo usermod -a -G wireshark $USER
```

### Virtual Machine Configuration

1. Power off the ISP machine
2. Configure a single NIC:
   - Go to Machine > Settings > Network
   - Disable all Adapters except Adapter 1
   - Set to either Bridged or NAT network (avoid NAT)
3. Create two linked clones named radius1 and radius2
   - Ensure MAC addresses are reinitialized

## Exercise 1: Basic Radius Server with Test Client

### Client Configuration

Edit `/etc/freeradius/3.0/clients.conf`:

```conf
client localhost {
    ipaddr = 127.0.0.1
    secret = testing123
    require_message_authenticator = no
    nas_type = other
}
```

### User Configuration

Edit `/etc/freeradius/3.0/users`:

```
"alice" Cleartext-Password := "password"
```

### Server Operation

1. Stop the FreeRADIUS service:
```bash
sudo service freeradius stop
```

2. Start FreeRADIUS in debug mode:
```bash
sudo freeradius -X -d /etc/freeradius/3.0
```

3. Test authentication:
```bash
echo "User-Name=alice, User-Password=password" | radclient 127.0.0.1 auth testing123 -x
```

## Exercise 2: HTTP Authentication with Apache

### Apache Configuration

1. Enable the RADIUS authentication module:
```bash
sudo a2enmod auth_radius
sudo service apache2 restart
```

2. Configure RADIUS settings in `/etc/apache2/ports.conf`:
```apache
AddRadiusAuth localhost:1812 testing123 5:3
AddRadiusCookieValid 1
```

3. Configure authentication for web pages in `/etc/apache2/sites-available/000-default.conf`:
```apache
<Directory /var/www/html>
    Options Indexes FollowSymLinks MultiViews
    AllowOverride None
    AuthType Basic
    AuthName "RADIUS Authentication for my site"
    AuthBasicProvider radius
    Require valid-user
</Directory>
```

4. Reload Apache configuration:
```bash
sudo service apache2 reload
```

### Testing

Test using either:
- Web browser: Navigate to http://localhost
- Command line: `curl --user alice:password http://localhost -v`

## Exercise 3: Roaming and Federation

### Configuration for radius1

Edit `/etc/freeradius/3.0/proxy.conf`:
```conf
home_server hs_domain_com {
    type = auth+acct
    ipaddr = $RADIUS2
    port = 1812
    secret = testing123
}

home_server_pool pool_domain_com {
    type = fail-over
    home_server = hs_domain_com
}

realm domain.com {
    pool = pool_domain_com
    nostrip
}
```

### Configuration for radius2

1. Configure realm in `/etc/freeradius/3.0/proxy.conf`:
```conf
realm domain.com {
}
```

2. Configure client in `/etc/freeradius/3.0/clients.conf`:
```conf
client $RADIUS1 {
    secret = testing123
}
```

3. Add user in `/etc/freeradius/3.0/users`:
```
"bob" Cleartext-Password := "password"
```

### Testing Roaming

Test using either:
- Web browser: Navigate to http://localhost and login with bob@domain.com
- Command line: `curl --user bob@domain.com:password http://localhost -v`

## Common Questions

1. **AVPs in Apache to Radius communication**: Use Wireshark with the radius filter to inspect the AVPs sent during Alice's login attempt.

2. **Proxy AVPs**: Additional AVPs are added to Access-Request messages when the local Radius server proxies requests to radius2.

3. **Adding Additional Domains**: To cover users from domain example.org on radius2, you would need to add a new realm configuration in the proxy.conf file similar to the domain.com configuration.

## Troubleshooting

If you encounter port conflicts when starting FreeRADIUS:
1. Stop background service: `sudo service freeradius stop`
2. For foreground instances: Use Ctrl+C in the terminal running the server