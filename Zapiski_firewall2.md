## Stateful Firewall Rules

### Prepare VirtualBox Image

1. **Start VirtualBox and change the MAC address of the machine network interface:**
   - Go to `Settings > Network > Adapter 1 > Advanced > MAC Address`
   - Generate a new random MAC address

2. **Change the Network Adapter:**
   - **Home/University Ethernet Network:**
     - `Settings > Network > Adapter 1 > Attached to: Bridged`
   - **Eduroam:**
     - Create a new NAT network:
       - `File > Preferences > Networks > NAT Networks > Add new NAT network`
       - Leave all settings to defaults
     - Set `Adapter 1` to use the NAT network that you have created previously

3. **Start the image and disable IPv6:**
   - Start the image and login as `isp/isp`
   - Since `iptables` supports IPv4 only, disable IPv6 to prevent packets from passing through that should be blocked
   - If you have not modified the `/etc/sysctl.conf` file in the previous exercise, complete this step in full. Otherwise, reload sysctl by running `sudo sysctl -p`

4. **To disable IPv6, open file `/etc/sysctl.conf` and add the following lines at the end of the file:**
   ```
   net.ipv6.conf.all.disable_ipv6 = 1
   net.ipv6.conf.default.disable_ipv6 = 1
   net.ipv6.conf.lo.disable_ipv6 = 1
   ```
   - Activate changes by running `sudo sysctl -p`
   - Verify that IPv6 has been disabled by running `cat /proc/sys/net/ipv6/conf/all/disable_ipv6`. This should output `1`

### Install Apache2 and SSH Server

1. **Install programs/packages that will be used for testing firewall rules:**
   - `sudo apt install openssh-server apache2 git curl`

2. **Generate default digital certificates for Apache2:**
   - `sudo make-ssl-cert generate-default-snakeoil --force-overwrite`

3. **Enable Apache2 SSL Site:**
   - `sudo a2ensite default-ssl`

4. **Enable Apache2 TLS/SSL module:**
   - `sudo a2enmod ssl`

5. **Restart Apache server:**
   - `sudo service apache2 restart`

6. **Check if Apache2 works:**
   - Open the web browser and visit `http://localhost` and `https://localhost`
   - Alternatively, install the `curl` package and test from within the terminal

7. **Check if SSH server works:**
   - Run `ssh localhost`, answer with `yes` and provide the password `isp`
   - Press `ctrl+d` to exit

### Download the Script Template

1. **Obtain the script template:**
   - Download the script manually or checkout the git repository: `git clone https://github.com/lem-course/isp-iptables.git`

2. **Change downloaded file's execution permissions:**
   - `chmod +x iptables2.sh`

### Solve Assignments (INPUT and OUTPUT Chains Only)

1. **Follow instructions in the script and solve assignments therein. For each task that you solve, test your solution and verify that it works. A typical cycle consists of the following steps:**
   - Write a solution
   - Start the script: `sudo ./iptables2.sh start`
   - Optionally, check which rules are activated: `sudo iptables --list -nv`
   - Test the rules by running the appropriate program:
     - ICMP with `ping`
     - DNS with `dig`, e.g., `dig www.fri.uni-lj.si`
     - HTTP with `curl`, e.g., `curl google.com`
     - SSH with `ssh`, e.g., `ssh isp@<ip-of-the-machine-you-are-connecting-to>`
   - Do not forget to restart the script when you add modifications: `sudo ./iptables2.sh restart`

## Firewall Forwarding Rules

So far we have been using `iptables` on computers that participate in the network as hosts (or end-nodes). But `iptables` can be used also on routers and thus protect entire networks behind them. This part of the lab session deals with setting firewall rules on a network router.

**Note:** The network setup for this assignment is fairly complicated. Read on carefully and if you get stuck, do not proceed until you have resolved the issue at hand.

1. **Shutdown the virtual image that you've been using so far.**

2. **In this part of the exercise we will use three virtual machines: a router, a client, and a server.**
   - The client will be located in the subnet, called `client_subnet`, and the server will be located in the subnet called `server_subnet`
   - The router will reside on both subnets and will route packages between those two subnets
   - Additionally, the router will have a third interface through which it will provide Internet connectivity

3. **Set up VirtualBox images:**
   - Create two additional virtual images by cloning the existing image. Name the first clone `client` and the second `server`. (We'll assume that the image you have been using so far is named `isp`)
   - You may create linked clones. Do not forget to generate new MAC addresses for the newly created images

4. **Configure the `isp` machine to use two additional network interface cards (NICs):**
   - `Machine > Settings > Network > Adapter 2`
   - Tick `Enable Network Adapter`, select `Internal Network` and put `client_subnet` in the `Name` field
   - Switch to tab `Adapter 3` and repeat the process, but this time name the `Internal Network` card `server_subnet`
   - The first network adapter on `isp` can be set to `NAT`, `Bridged Adapter`, or `NAT Network`. It does not really matter which, as long as it provides Internet connectivity. Confirm the changes by clicking `OK`

5. **Configure the NIC on the client:**
   - Set its network interface to `Internal Network` and select `client_subnet` as the name

6. **Configure the NIC on the server:**
   - Set its network interface to `Internal Network` and select `server_subnet` as the name

### Prepare Router Machine (isp)

1. **Start the `isp` machine:**
   - Notice that the machine has three NIC cards: run `ip addr` and observe `enp0s3`, `enp0s8`, and `enp0s9`
   - Only `enp0s3` managed to obtain an IP address, while `enp0s8` and `enp0s9` did not. The reason is that the subnets which `enp0s8` and `enp0s9` connect to do not have DHCP servers. This means that we'll have to set up IPs manually

2. **Assign IPs to `isp` machine for `enp0s8` and `enp0s9`:**
   - Since the `client_subnet` uses addresses from `10.0.0.0/24` and `server_subnet` addresses from `172.16.0.0/24`, we'll use the first available address that comes to mind: `10.0.0.1` for `enp0s8` and `172.16.0.1` for `enp0s9`

3. **Open file `/etc/netplan/01-network-manager-all.yaml` and change the contents to the following:**
   ```
   network:
     version: 2
     ethernets:
       enp0s3:
         dhcp4: true
         dhcp-identifier: mac
       enp0s8:
         addresses: [10.0.0.1/24]
       enp0s9:
         addresses: [172.16.0.1/24]
   ```
   - Apply these changes by running `sudo netplan apply`
   - Confirm that the addresses have been successfully set by running `ip addr`

4. **Enable routing for IPv4 so that the `isp` will actually behave as a proper router:**
   - `echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward`

5. **To route internet-bound traffic from client and server subnets, configure the `isp` to act as a network address translator (NAT):**
   - `sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE`

### Prepare the Client Machine

1. **Configuring the client and the server is simpler. We have to do three things:**
   - Assign them IP addresses (`10.0.0.2`)
   - DNS servers (`8.8.8.8`)
   - Instruct them to send packets through the `isp` (`10.0.0.1`) machine

2. **Open file `/etc/netplan/01-network-manager-all.yaml` and change the contents to the following:**
   ```
   network:
     version: 2
     ethernets:
       enp0s3:
         # assign the IP address
         addresses: [10.0.0.2/24]
         # set the default route through isp
         routes:
           - to: default
             via: 10.0.0.1
         # use Google's DNS
         nameservers:
           addresses: [8.8.8.8]
   ```
   - Apply these changes by running `sudo netplan apply`
   - You should now be able to ping the `isp` and any host on the Internet through `isp`; try pinging `8.8.8.8` which is a DNS server from Google

### Prepare the Server Machine

1. **Configuring the server is almost identical to configuring the client. There are only two differences:**
   - The server should get the IP address of `172.16.0.2` and not `10.0.0.2`
   - While the `isp` retains its role of the router, the default route on the server should point to `172.16.0.1` and not to `10.0.0.1`

2. **After setting these values, you should be able to set up arbitrary connections between hosts in both subnets as well as the Internet and the router.**

### Filtering

1. **Edit `iptables2.sh` script and add entries to the FORWARD chain that permit ICMP, DNS, SSH, HTTP, and HTTPS traffic.**

2. **When you test these rules, make sure that you launch requests from the client machine. You can test connectivity between programs running on hosts in the `server_subnet`, `client_subnet`, router (the `isp` machine), and other hosts on the public Internet.**

### Additional Tasks

1. **Allow all SSH connections between `client_subnet` and the `server_subnet`. At the same time, prevent SSH connections to the public Internet.**

2. **On the router, prevent any access to `facebook.com`.**
   - **Hints:**
     - To find out the required IP address, you may use the following command: `dig +noall +answer facebook.com | cut -f6 | xargs | tr " " ,`
     - Save the result of this command into a variable, and use that variable as the destination IP address in the `iptables` rule
     - Make sure that this rule gets evaluated before any other rule that might accept HTTP traffic

3. **Limit the number of ping requests to the firewall to 10 per minute when they come from the public Internet.**