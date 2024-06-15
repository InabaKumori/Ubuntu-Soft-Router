# Ubuntu SoftRouter: A Comprehensive Guide
## Introduction
Welcome to the Ubuntu SoftRouter repository! This guide aims to provide a detailed walkthrough on setting up an Ubuntu-based router without a GUI, similar to OpenWrt. By following this tutorial, you'll be able to transform your Ubuntu Server into a powerful and customizable router solution.
## Table of Contents
1. [Hardware Requirements](#hardware-requirements)
2. [Installing Ubuntu Server](#installing-ubuntu-server)
3. [Initial Configuration](#initial-configuration)
4. [Disabling NetworkManager and Configuring Netplan](#disabling-networkmanager-and-configuring-netplan)
5. [Installing and Configuring Required Packages](#installing-and-configuring-required-packages)
6. [Configuring Firewall Rules](#configuring-firewall-rules)
7. [Starting Services](#starting-services)
8. [Testing and Verification](#testing-and-verification)
9. [Advanced Configuration](#advanced-configuration)
10. [Troubleshooting](#troubleshooting)
11. [Contributing](#contributing)
12. [License](#license)
## Hardware Requirements
Before getting started, ensure that you have the following hardware:
- A computer or a single-board computer (e.g., Raspberry Pi) with multiple network interfaces
- Ethernet cables for each network interface
- A USB drive or CD/DVD for installing Ubuntu Server
## Installing Ubuntu Server
Follow these steps to install Ubuntu Server on your hardware:
1. Download the latest version of Ubuntu Server from the official website (https://ubuntu.com/download/server).
2. Create a bootable USB drive or burn the ISO image to a CD/DVD.
3. Connect the computer to a monitor, keyboard, and power source.
4. Insert the bootable USB drive or CD/DVD and boot from it.
5. Follow the installation wizard to install Ubuntu Server. Choose the "minimal" installation option for a lightweight setup.
6. Set up a username and password for the administrator account.
7. Once the installation is complete, remove the installation media and reboot the system.
## Initial Configuration
After installing Ubuntu Server, perform the initial configuration steps:
1. Log in to the Ubuntu Server using the administrator account.
2. Update the system packages by running the following commands:
   ```
   sudo apt update
   sudo apt upgrade
   ```
## Disabling NetworkManager and Configuring Netplan
By default, Ubuntu uses NetworkManager for network configuration. However, for a router setup, it's recommended to use Netplan. Follow these steps to disable NetworkManager and configure Netplan:
1. Disable NetworkManager by running the following commands:
   ```
   sudo systemctl stop NetworkManager
   sudo systemctl disable NetworkManager
   ```
2. Identify the network interfaces on your system by running the following command:
   ```
   ip addr
   ```
   Take note of the interface names (e.g., enp5s0, enp7s0f0, enp7s0f1, etc.).
3. Configure the network interfaces by editing the `/etc/netplan/00-installer-config.yaml` file. Assign static IP addresses to the LAN interfaces and configure the WAN interface to use DHCP:
   ```yaml
   network:
     version: 2
     renderer: networkd
     ethernets:
       enp5s0:
         dhcp4: true
         dhcp6: true
         optional: true
         accept-ra: true
       enp7s0f0:
         dhcp4: false
         dhcp6: false
         optional: true
       enp7s0f1:
         dhcp4: false
         dhcp6: false
         optional: true
       enp7s0f2:
         dhcp4: false
         dhcp6: false
         optional: true
       enp7s0f3:
         dhcp4: false
         dhcp6: false
         optional: true
     bridges:
       br_lan: # A bridged gateway named br_lan combining several network interfaces
         interfaces: # Contains the unique identifiers of each network interfaces 
           - "enp7s0f0"
           - "enp7s0f1"
           - "enp7s0f2"
           - "enp7s0f3"
         addresses: # Local IP addresses 
           - 192.168.100.1/24
           - fc00:192:168:2::1/64
         dhcp4: false # Disabling DHCPv4
         dhcp6: false # Disabling DHCPv6
         parameters:
           stp: true # Prevent internal looping
           forward-delay: 4
         optional: true # optional for network optimization
   ```
   Replace the interface names and IP addresses according to your network setup.
4. Apply the network configuration by running:
   ```
   sudo netplan apply
   ```
## Installing and Configuring Required Packages
Install and configure the necessary packages for routing and firewall functionality:
1. Install the required packages:
   ```
   sudo apt install iptables dnsmasq hostapd
   ```
2. Configure dnsmasq for DHCP and DNS services. Edit the `/etc/dnsmasq.conf` file and add the following lines:
   ```
   interface=br_lan
   dhcp-range=192.168.100.100,192.168.100.200,255.255.255.0,24h
   dhcp-option=3,192.168.100.1
   dhcp-option=6,192.168.100.1
   
   dhcp-range=fc00:192:168:2::1,fc00:192:168:2::ff,24h
   dhcp-range=fc00:192:168:2::,slaac
   dhcp-option=option6:dns-server,[fc00:192:168:2::1]
   enable-ra
   dhcp-authoritative
   
   cache-size=0

   # Please set the DNS manually and uncomment the following if you have mosdns properly set up.
   # server=127.0.0.1#5335
   # Otherwise please uncomment the following line in order to fetch DNS responses
   # server=8.8.8.8
   dns-forward-max=10000
   no-resolv
   no-poll
   port=53
   ```
   This configures dnsmasq to provide DHCP leases on each LAN interface with the specified IP ranges and lease times.
3. Configure hostapd for wireless access point functionality (if required). Create a new file `/etc/hostapd/hostapd.conf` and add the following lines:
   ```
   interface=wlan0
   driver=nl80211
   ssid=YourWiFiNetworkName
   hw_mode=g
   channel=7
   macaddr_acl=0
   auth_algs=1
   ignore_broadcast_ssid=0
   wpa=2
   wpa_passphrase=YourWiFiPassword
   wpa_key_mgmt=WPA-PSK
   wpa_pairwise=TKIP
   rsn_pairwise=CCMP
   ```
   Replace `wlan0` with the actual wireless interface name, `YourWiFiNetworkName` with your desired Wi-Fi network name, and `YourWiFiPassword` with a secure password.
4. Enable packet forwarding by uncommenting the following line in the `/etc/sysctl.conf` file:
   ```
   net.ipv4.ip_forward=1
   net.ipv4.conf.all.forwarding=1
   net.ipv4.conf.default.forwarding=1
   net.ipv4.conf.all.route_localnet=1
   net.ipv6.conf.all.forwarding=1
   net.ipv6.conf.default.forwarding=1
   ```
   Apply the changes by running:
   ```
   sudo sysctl -p
   ```
   or if changes are not applied:
   ```
   systemctl restart systemd-sysctl.service
   ```
## Configuring Firewall Rules
Set up firewall rules to control network traffic:
1. Create a script to configure the firewall rules. Create a new file `/etc/init.d/firewall` with the following content:
   ```bash
   #!/bin/sh
   ### BEGIN INIT INFO
   # Provides:          firewall
   # Required-Start:    $remote_fs $syslog
   # Required-Stop:     $remote_fs $syslog
   # Default-Start:     2 3 4 5
   # Default-Stop:      0 1 6
   # Short-Description: Start firewall at boot time
   # Description:       Enable service provided by firewall.
   ### END INIT INFO
   
   # Clear existing rules
   iptables -F
   iptables -X
   iptables -t nat -F
   iptables -t nat -X
   iptables -t mangle -F
   iptables -t mangle -X
   
   # Set default policies
   iptables -P INPUT ACCEPT
   iptables -P FORWARD ACCEPT
   iptables -P OUTPUT ACCEPT
   
   WAN_NAME='enp5s0'
   
   iptables -t nat -N mt_rtr_4_n_rtr
   iptables -t nat -A POSTROUTING -j mt_rtr_4_n_rtr
   iptables -t nat -A mt_rtr_4_n_rtr -o ${WAN_NAME} -j MASQUERADE 
   
   iptables -t mangle -N mt_rtr_4_m_rtr
   iptables -t mangle -A FORWARD -j mt_rtr_4_m_rtr
   iptables -t mangle -A mt_rtr_4_m_rtr -o ${WAN_NAME} -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu 
   iptables -t mangle -A mt_rtr_4_m_rtr -m state --state RELATED,ESTABLISHED -j ACCEPT 
   iptables -t mangle -A mt_rtr_4_m_rtr -m conntrack --ctstate INVALID -j DROP
   iptables -t mangle -A mt_rtr_4_m_rtr -p tcp -m tcp ! --tcp-flags FIN,SYN,RST,ACK SYN -m state --state NEW -j DROP
   iptables -t mangle -A mt_rtr_4_m_rtr -p tcp -m tcp --tcp-flags FIN,SYN,RST,PSH,ACK,URG FIN,SYN,RST,PSH,ACK,URG -j DROP
   iptables -t mangle -A mt_rtr_4_m_rtr -p tcp -m tcp --tcp-flags FIN,SYN,RST,PSH,ACK,URG NONE -j DROP
   iptables -t mangle -A mt_rtr_4_m_rtr -i br_lan -o ${WAN_NAME} -j ACCEPT
   
   # Allow all incoming traffic
   iptables -A INPUT -j ACCEPT
   
   # Allow all forwarded traffic
   iptables -A FORWARD -j ACCEPT
   
   # Save the rules
   iptables-save > /etc/iptables.rules
   
   # Clear existing rules
   ip6tables -F
   ip6tables -X
   ip6tables -t nat -F
   ip6tables -t nat -X
   ip6tables -t mangle -F
   ip6tables -t mangle -X
   
   # Set default policies
   ip6tables -P INPUT ACCEPT
   ip6tables -P FORWARD ACCEPT
   ip6tables -P OUTPUT ACCEPT
   
   ip6tables -t nat -N mt_rtr_6_n_rtr
   ip6tables -t nat -A POSTROUTING -j mt_rtr_6_n_rtr
   ip6tables -t nat -A mt_rtr_6_n_rtr -o ${WAN_NAME} -j MASQUERADE # 添加路由到作为WAN的网卡的自动源地址转换规则
   
   ip6tables -t mangle -N mt_rtr_6_m_rtr
   ip6tables -t mangle -A FORWARD -j mt_rtr_6_m_rtr
   ip6tables -t mangle -A mt_rtr_6_m_rtr -o ${WAN_NAME} -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
   ip6tables -t mangle -A mt_rtr_6_m_rtr -m state --state RELATED,ESTABLISHED -j ACCEPT
   ip6tables -t mangle -A mt_rtr_6_m_rtr -m conntrack --ctstate INVALID -j DROP
   ip6tables -t mangle -A mt_rtr_6_m_rtr -p tcp -m tcp ! --tcp-flags FIN,SYN,RST,ACK SYN -m state --state NEW -j DROP
   ip6tables -t mangle -A mt_rtr_6_m_rtr -p tcp -m tcp --tcp-flags FIN,SYN,RST,PSH,ACK,URG FIN,SYN,RST,PSH,ACK,URG -j DROP
   ip6tables -t mangle -A mt_rtr_6_m_rtr -p tcp -m tcp --tcp-flags FIN,SYN,RST,PSH,ACK,URG NONE -j DROP
   ip6tables -t mangle -A mt_rtr_6_m_rtr -i br_lan -o ${WAN_NAME} -j ACCEPT
   
   # Allow all incoming traffic
   ip6tables -A INPUT -j ACCEPT
   
   # Allow all forwarded traffic
   ip6tables -A FORWARD -j ACCEPT
   
   # Save the rules
   ip6tables-save > /etc/ip6tables.rules
   ```
   Make the script executable by running:
   ```
   sudo chmod +x /etc/init.d/firewall
   ```
2. Configure the firewall script to run at system startup. Create a new file `/etc/systemd/system/firewall.service` with the following content:
   ```
   [Unit]
   Description=Firewall Service
   After=network.target
   [Service]
   ExecStart=/etc/init.d/firewall
   [Install]
   WantedBy=multi-user.target
   ```
   Enable the firewall service by running:
   ```
   sudo systemctl enable firewall.service
   ```
## Starting Services
Start the necessary services for the router functionality:
1. Start the dnsmasq service:
   ```
   sudo systemctl start dnsmasq
   ```
2. Start the hostapd service (if configured):
   ```
   sudo systemctl start hostapd
   ```
3. Reboot the system to ensure all changes take effect:
   ```
   sudo reboot
   ```

## Enabling link-tracing module (optional)
1. Add the following line for the file located in /etc/modules-load.d/modules.conf
   ```
   nf_conntrack
   ```
2. reboot
## Testing and Verification
Verify that your Ubuntu-based router is functioning correctly:
1. Connect client devices to each LAN interface or the Wi-Fi network.
2. Verify that the client devices receive IP addresses from the DHCP server.
3. Test internet connectivity by accessing a website or running a ping test.
4. Verify that the firewall rules are applied by checking the iptables rules:
   ```
   sudo iptables -L -n -v
   ```

## Disable IPv6 for V2rayA (Optional)
1. Modify the startup script
   ```
   vim /usr/lib/systemd/system/v2raya.service
   ```
2. Add argument --ipv6-support=off
   ```
   ExecStart=/usr/bin/v2raya --log-disable-timestamp --ipv6-support=off
   ```
3. Restart service
   ```
   systemctl reenable v2raya
   systemctl restart v2raya
   ```

## Bind network interfaces with MAC addresses to their corresponding names, drivers and other settings.
1. Modifythe following config file with similar styles.
   ```
   # Located in /etc/udev/rules.d/custom-devices.rules
   # Network Interface Card
   # Format: SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="(replace with MAC address)", ATTR{dev_id}=="0x0", ATTR{type}=="1", NAME="(replace with desired interface name)"
   
   # Realtek Semiconductor Co., Ltd. RTL8111/8168/8411 PCI Express Gigabit Ethernet Controller
   SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="40:b0:76:42:58:96", ATTR{dev_id}=="0x0", ATTR{type}=="1", NAME="enp5s0"
   
   # Broadcom Inc. and subsidiaries NetXtreme BCM5719 Gigabit Ethernet PCIe (Port 1)
   SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="40:a8:f0:25:34:a8", ATTR{dev_id}=="0x0", ATTR{type}=="1", NAME="enp7s0f0"
   
   # Broadcom Inc. and subsidiaries NetXtreme BCM5719 Gigabit Ethernet PCIe (Port 2)
   SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="40:a8:f0:25:34:a9", ATTR{dev_id}=="0x0", ATTR{type}=="1", NAME="enp7s0f1"
   
   # Broadcom Inc. and subsidiaries NetXtreme BCM5719 Gigabit Ethernet PCIe (Port 3)
   SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="40:a8:f0:25:34:aa", ATTR{dev_id}=="0x0", ATTR{type}=="1", NAME="enp7s0f2"
   
   # Broadcom Inc. and subsidiaries NetXtreme BCM5719 Gigabit Ethernet PCIe (Port 4)
   SUBSYSTEM=="net", ACTION=="add", DRIVERS=="?*", ATTR{address}=="40:a8:f0:25:34:ab", ATTR{dev_id}=="0x0", ATTR{type}=="1", NAME="enp7s0f3"
   ```
   
## Advanced Configuration
Here are some additional configuration options to enhance your Ubuntu-based router:
- **VPN Server**: Set up a VPN server to securely access your network remotely. You can use OpenVPN or WireGuard for this purpose.
- **DNS Over HTTPS (DoH)**: Implement DNS over HTTPS to encrypt DNS queries and enhance privacy. You can use tools like dnscrypt-proxy or cloudflared.
- **Quality of Service (QoS)**: Configure QoS rules to prioritize network traffic based on your requirements. You can use tools like tc (Traffic Control) and htb (Hierarchical Token Bucket) for QoS management.
- **Dynamic DNS**: If you have a dynamic IP address, set up a dynamic DNS service to access your router using a domain name. Popular dynamic DNS providers include No-IP, DynDNS, and Duck DNS.
- **Port Forwarding**: Configure port forwarding rules to allow incoming connections to specific services running on your network. This is useful for hosting servers or accessing devices remotely.
- **Logging and Monitoring**: Set up logging and monitoring solutions to keep track of network activity and detect any anomalies. Tools like Nagios, Zabbix, or Prometheus can be used for monitoring purposes.
Refer to the respective documentation and guides for each advanced configuration option to implement them on your Ubuntu-based router.
## Troubleshooting
If you encounter any issues while setting up or using your Ubuntu-based router, consider the following troubleshooting steps:
- Check the system logs for any error messages or warnings. You can use the `journalctl` command or view the log files in the `/var/log` directory.
- Verify that all the required services (e.g., dnsmasq, hostapd) are running correctly. Use the `systemctl status` command to check their status.
- Double-check the configuration files for any syntax errors or incorrect settings.
- Ensure that the firewall rules are properly configured and not blocking necessary traffic.
- Consult the official Ubuntu Server documentation (https://help.ubuntu.com/lts/serverguide/) or seek support from the Ubuntu community forums (https://ubuntuforums.org/).
## Contributing
Contributions to this repository are welcome! If you have any improvements, bug fixes, or additional features to suggest, please follow these steps:
1. Fork the repository.
2. Create a new branch for your feature or bug fix.
3. Make your modifications and test thoroughly.
4. Commit your changes and push them to your forked repository.
5. Submit a pull request, describing your changes and their benefits.
Please ensure that your contributions adhere to the repository's coding style and guidelines.
## License
This repository is licensed under the GNU General Public License v3.0. You are free to use, modify, and distribute the code in this repository, subject to the terms and conditions of the license. For more information, please refer to the [LICENSE](LICENSE) file.
