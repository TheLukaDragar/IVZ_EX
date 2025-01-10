

# SSH Lab Guide

## Preparation

Start the base image and install the required software. We will require the following packages:
- openssh-server
- openssh-client
- wireshark
- apache2
- curl

Update the package list and install required software:
```bash
sudo apt update
sudo apt install openssh-server openssh-client wireshark apache2 curl
```

*Note: While installing wireshark, when asked "Should non-superusers be able to capture packets?" select yes. If you select no by mistake, you can change your selection by running:*
```bash
sudo dpkg-reconfigure wireshark-common
```

Add your user to the wireshark group:
```bash
sudo usermod -a -G wireshark $USER
```

Shutdown the virtual machine and clone the base image twice (use linked clone and regenerate the MAC address). Name the new machines `ssh-server` and `ssh-client`.

### Network Configuration
- Configure both machines to use a single network interface card (NIC)
- Disable Adapter 2, Adapter 3 and Adapter 4 in Machine > Settings > Network
- Ensure Adapter 1 is enabled
- Place both machines either on Bridged or in the same NAT network
- Verify machines can ping each other

## Machine ssh-server Setup

### Change Hostname
1. Open `/etc/hosts` and add:
```
127.0.1.1 ssh-server
```
2. Run:
```bash
sudo hostnamectl set-hostname ssh-server
```
3. Restart the terminal. Lines should now start with `isp@ssh-server`

### Regenerate SSH Server Keys
*Note: Provide an empty passphrase when asked*

```bash
sudo ssh-keygen -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key
sudo ssh-keygen -t rsa   -f /etc/ssh/ssh_host_rsa_key
sudo ssh-keygen -t dsa   -f /etc/ssh/ssh_host_dsa_key
sudo ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key
```

## Assignments

*Assume `$CLIENT` is the IP of ssh-client and `$SERVER` is the IP of ssh-server*

### 1. Username/Password Client Authentication, Server Authentication

1. On ssh-client, connect to server:
```bash
ssh isp@$SERVER
```

2. To verify server's public key fingerprint on ssh-server, use:
```bash
# For ECDSA key
ssh-keygen -lf /etc/ssh/ssh_host_ecdsa_key.pub
# For ED25519 key
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
# For RSA key
ssh-keygen -lf /etc/ssh/ssh_host_rsa_key.pub
# For DSA key
ssh-keygen -lf /etc/ssh/ssh_host_dsa_key.pub
```

**Question 1**: Verify the displayed fingerprint is correct. What kind of attack could be taking place if fingerprints mismatch?

3. Change SSH keypairs on ssh-server:
```bash
sudo ssh-keygen -t ecdsa -f /etc/ssh/ssh_host_ecdsa_key
sudo ssh-keygen -t rsa -f /etc/ssh/ssh_host_rsa_key
sudo ssh-keygen -t dsa -f /etc/ssh/ssh_host_dsa_key
sudo ssh-keygen -t ed25519 -f /etc/ssh/ssh_host_ed25519_key
```

### 2. Client Public Key Authentication

On ssh-client, generate SSH keys:
```bash
ssh-keygen -t rsa
ssh-keygen -t dsa
ssh-keygen -t ecdsa
```

Connect using public key:
```bash
ssh -i ~/.ssh/id_rsa isp@$SERVER
```

Copy public key to server:
```bash
ssh-copy-id isp@$SERVER
```

Disable password authentication on ssh-server:
1. Edit `/etc/ssh/sshd_config`:
```
PasswordAuthentication no
```
2. Restart SSH service:
```bash
sudo service ssh restart
```

Test password authentication is disabled:
```bash
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no $SERVER
```

### 3. Tunneling with SSH

Configure Apache on ssh-server for localhost only:
1. Edit `/etc/apache2/sites-available/000-default.conf`:
```apache
<Directory /var/www/html>
    Require ip 127.0.0.1/8
</Directory>
```
2. Reload Apache:
```bash
sudo service apache2 reload
```

Create SSH tunnel:
```bash
ssh -L 127.0.0.1:8080:127.0.0.1:80 -N $SERVER
```

**Question 2**: Observe Apache access log:
```bash
tail -f /var/log/apache2/access.log
```

### 4. Reverse SSH Tunneling

Configure IPv6 and firewall on ssh-server:

1. Add to `/etc/sysctl.conf`:
```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

2. Apply changes:
```bash
sudo sysctl -p
```

3. Configure iptables:
```bash
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A INPUT  -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A OUTPUT  -p tcp --dport 22 -m state --state NEW -j ACCEPT
```

Create reverse tunnel:
```bash
ssh -R 127.0.0.1:8080:127.0.0.1:80 -N isp@$CLIENT
```

## Additional Tasks

1. Use Wireshark to observe message exchange during communication setup

2. Explore file transfer commands:
- `scp` - secure copy (remote file copy program)
- `rsync` - fast, versatile, remote (and local) file-copying tool