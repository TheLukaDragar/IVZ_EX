I'll help format this document into clear, readable markdown while preserving all the information.



# Setting Up a Company Network Infrastructure with VPN

## Introduction
As part of setting up a company network infrastructure, we'll set up a tunneled VPN between the headquarters and its remote branch. We'll be using StrongSwan, an open-source implementation of IKE. StrongSwan is an IKE keying daemon that sets up ISAKMP and IKE associations between various network points.

### Requirements
- Four virtual machines:
  - Two routers (with two network interfaces each)
  - Two hosts (with one network interface each)

## Initial Setup

### Preparing the ISP Machine
1. Start the ISP machine and install required packages:
```bash
sudo apt update
sudo apt install strongswan strongswan-pki libcharon-extra-plugins apache2 wireshark
```

**Note:** During Wireshark installation, select "yes" when asked "Should non-superusers be able to capture packets?" Add your user to the wireshark group:
```bash
sudo usermod -a -G wireshark $USER
```

2. Configure Network Interface Cards (NICs):
   - Go to Machine > Settings > Network
   - Set Adapter 1 to NAT Network
   - Set Adapter 2 to Internal-Network

### Creating Virtual Machines
Clone the ISP machine four times with the following configurations:

1. **hq_router**:
   - Adapter 1: NAT Network
   - Adapter 2: Internal-Network (hq_subnet)

2. **branch_router**:
   - Adapter 1: NAT Network
   - Adapter 2: Internal-Network (branch_subnet)

3. **hq_server**:
   - Adapter 1: Internal-Network (hq_subnet)
   - Adapter 2: Disabled

4. **branch_client**:
   - Adapter 1: Internal-Network (branch_subnet)
   - Adapter 2: Disabled

## Network Configuration

### Headquarters Setup

#### HQ Router Configuration
1. Edit `/etc/netplan/01-network-manager-all.yaml`:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp-identifier: mac
    enp0s8:
      addresses: [10.1.0.1/16]
```

2. Apply changes and enable packet forwarding:
```bash
sudo netplan apply
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

#### HQ Server Configuration
1. Edit `/etc/netplan/01-network-manager-all.yaml`:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [10.1.0.2/16]
      routes:
        - to: default
          via: 10.1.0.1
      nameservers:
        addresses: [8.8.8.8]
```

### Branch Setup

#### Branch Router Configuration
1. Edit `/etc/netplan/01-network-manager-all.yaml`:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp-identifier: mac
    enp0s8:
      addresses: [10.2.0.1/16]
```

2. Apply changes and enable packet forwarding:
```bash
sudo netplan apply
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

#### Branch Client Configuration
1. Edit `/etc/netplan/01-network-manager-all.yaml`:
```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [10.2.0.2/16]
      routes:
        - to: default
          via: 10.2.0.1
      nameservers:
        addresses: [8.8.8.8]
```

### Checkpoint Verification
Verify the following connectivity:
- Ping between hq_router and hq_server (10.1.0.0/16 network)
- Ping between branch_router and branch_client (10.2.0.0/16 network)
- Ping between hq_router and branch_router (using public addresses on enp0s3)

## VPN IPsec Tunnel Setup

### HQ Router VPN Configuration
1. Edit `/etc/ipsec.conf`:
```conf
config setup

conn %default
        ikelifetime=60m
        keylife=20m
        rekeymargin=3m
        keyingtries=1
        keyexchange=ikev2
        authby=secret

conn net-net
        leftsubnet=10.1.0.0/16
        leftfirewall=yes
        leftid=@hq
        right=$BRANCH_IP
        rightsubnet=10.2.0.0/16
        rightid=@branch
        auto=add
```

2. Edit `/etc/ipsec.secrets`:
```
@hq @branch : PSK "secret"
```

3. Restart IPsec:
```bash
sudo ipsec restart
```

### Branch Router VPN Configuration
1. Edit `/etc/ipsec.conf`:
```conf
config setup

conn %default
        ikelifetime=60m
        keylife=20m
        rekeymargin=3m
        keyingtries=1
        keyexchange=ikev2
        authby=secret


conn net-net
        leftsubnet=10.2.0.0/16
        leftid=@branch
        leftfirewall=yes
        right=$HQ_IP
        rightsubnet=10.1.0.0/16
        rightid=@hq
        auto=add
        #leftauth="psk"
        #rightauth="psk"
```



2. Edit `/etc/ipsec.secrets`:
```
@hq @branch : PSK "secret"
```

3. Restart IPsec:
```bash
sudo ipsec restart
```

## Establishing the VPN Link
1. Check IPsec status:
```bash
sudo ipsec status[all]
```

2. Establish tunnel (on either router):
```bash
sudo ipsec up net-net
```

For debugging:
```bash
sudo ipsec start --nofork
```

## Lab Exercises

### Exercise 1: Traffic Analysis
1. Run Wireshark on either router
2. Observe ISAKMP, ICMP, and ESP traffic using filter: `isakmp || esp || icmp`
3. Restart StrongSwan and observe:
   - Security Association establishment (IKE PHASE 1)
   - Key Exchange (IKE PHASE 2)
   - Monitor logs: `tail -f -n 0 /var/log/auth.log`
4. Check Security Policy Database:
```bash
sudo ip xfrm policy
```

### Questions

1. **Question 1:** Examine SPIs using `sudo ip xfrm state`. Why are there two SPIs?

2. **Question 2:** Why can't hq_server and branch_client access the Internet? How can this be fixed?

3. **Question 3:** Analyze the output of `mtr 10.2.0.2` on hq_server. How would this change if the routers were 10 network hops apart?

### Additional Configuration
To modify cipher suites:
1. Check current suites: `sudo ipsec statusall`
2. Modify `/etc/ipsec.conf` on both routers to use AES_GCM_16_256
3. Test with: `ping -c 3 10.2.0.1`