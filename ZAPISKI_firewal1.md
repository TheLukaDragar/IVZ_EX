# Stateless Firewall Rules Setup Guide

## VirtualBox Configuration
1. Start VirtualBox and configure the network interface:
   - Go to Settings > Network > Adapter 1 > Advanced > MAC Address
   - Generate a new random MAC address

2. Configure Network Adapter based on your network:
   - **Home/University Ethernet Network**: 
     - Settings > Network > Adapter 1 > Attached to: Bridged
   - **Eduroam**: 
     - Create new NAT network:
       1. File > Preferences > Networks > NAT Networks
       2. Add new NAT network (use default settings)
       3. Set Adapter 1 to use the new NAT network

## Ubuntu Image Configuration

### Initial Setup
1. Login as `isp/isp`
2. Disable IPv6 by adding to `/etc/sysctl.conf`:
   ```bash
   net.ipv6.conf.all.disable_ipv6 = 1
   net.ipv6.conf.default.disable_ipv6 = 1
   net.ipv6.conf.lo.disable_ipv6 = 1
   ```
3. Activate changes: `sudo sysctl -p`
4. Verify IPv6 disabled: `cat /proc/sys/net/ipv6/conf/all/disable_ipv6` (should output 1)

### Package Installation
```bash
sudo apt-get install openssh-server apache2 curl git
```

### Apache2 Configuration
1. Generate SSL certificates:
   ```bash
   sudo make-ssl-cert generate-default-snakeoil --force-overwrite
   ```
2. Enable SSL:
   ```bash
   sudo a2ensite default-ssl
   sudo a2enmod ssl
   sudo service apache2 restart
   ```
3. Test Apache:
   - Visit `http://localhost` and `https://localhost`
   - Or use curl to test

4. Test SSH: `ssh localhost` (answer 'yes' and use password 'isp')

## Cloning the Image
1. Shutdown guest: `sudo poweroff`
2. Clone process:
   - Right-click image in VirtualBox → Clone (Ctrl+O)
   - Choose Expert mode
   - Name the clone (e.g., isp-2)
   - Select "Linked clone"
   - Enable "Reinitialize MAC address of all network cards"
   - Click Clone

## Running the Images
1. Start both images
2. Run `sudo sysctl -p` on both to disable IPv6
3. Test connectivity:
   - Get IP addresses: `ip addr`
   - Test connection: `ping <ip_addr>`

## Script Setup
1. Get the template:
   ```bash
   git clone https://github.com/lem-course/isp-iptables.git
   ```
2. Set permissions:
   ```bash
   chmod +x iptables1.sh
   ```

## Testing Process
1. Start firewall rules:
   ```bash
   sudo ./iptables1.sh start
   ```
2. List active rules:
   ```bash
   sudo iptables --list -vn
   ```

### Testing Tools
- ICMP: `ping`
- DNS: `dig` (e.g., `dig www.fri.uni-lj.si`)
- HTTP: `curl` (e.g., `curl google.com`)
- SSH: `ssh isp@<target-ip>`

### Testing Cycle
1. Solve task
2. Start/restart firewall rules
3. Inspect active rules
4. Test with appropriate program
5. Reset rules: `sudo ./iptables1.sh reset`

## Resources
- [Linux Tutorial - iptables Network Gateway](http://www.yolinux.com/TUTORIALS/LinuxTutorialIptablesNetworkGateway.html)
- [Linux Manual](http://faculty.ucr.edu/~tgirke/Documents/UNIX/linux_manual.html)