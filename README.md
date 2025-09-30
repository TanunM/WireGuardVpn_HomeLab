# WireGuardVPN_HomeLab

## Introduction
This document provides a detailed guide for establishing a basic WireGuard VPN tunnel using an Ubuntu Server as the VPN host and a Windows machine as the VPN client. The lab focuses on setting up a persistent, peer-to-peer connection that utilizes Network Address Translation (NAT) on the server to route client traffic securely to the internet, effectively masking the client's original public IP address.

## Objective
The objective of this lab is to:
* Gain hands-on experience installing and configuring the WireGuard VPN software on a Linux server.
* Successfully configure public and private keys to establish a secure peer-to-peer tunnel.
* Implement IP forwarding and iptables NAT/Masquerade rules to enable the server to route client internet traffic.
* Ensure all configurations (Interface, Keys, IP routing, and NAT) are persistent across server reboots using systemd and iptables-persistent.
* Verify that the Windows client's traffic is correctly exiting the network via the Ubuntu server's IP address.

##  Requirements and Tools

| Category | Requirements | Tools |
| :--- | :--- | :--- |
| **Server** | Ubuntu Server LTS (VM or Cloud Instance) | SSH Client |
| **Client** | Windows 10/11 OS | WireGuard for Windows |
| **System** | RAM: 1 GB, Disk Space: 10 GB free | Virtualization Software (VirtualBox, VMware) |
| **Networking** | Server must have a public-facing network interface. | wg, ip, iptables |

## Step 1: Initial Server Setup and WireGuard Installation in Ubuntu Server

### 1.1 Install Ubuntu and Connect via SSH
1. Install Ubuntu Server LTS in your VM.
2. Access the server remotely using SSH:
```bash
ssh <username>@<server IP>
```
3. Update and Upgrade System
```bash
sudo apt update && sudo apt upgrade
```
4. Install WireGuard
```bash
sudo apt install wireguard -y
```

### 1.2 Generate Keys
1. Generate the private key in the user's home directory:
```bash
wg genkey > private
```
2. Set restrictive permissions on the private key file:
```bash
sudo chmod 600 ./private
```
<img src='https://github.com/TanunM/WireGuardVpn_HomeLab/blob/main/gallery/generate_private_key.png'/>

3. Generate the corresponding public key:
```bash
wg pubkey < private > public
```

## Step 2: Configuring the WireGuard Interface

### 2.1 Create the WireGuard Interface
1. Add the virtual WireGuard interface named wg0:
``` bash
sudo ip link add wg0 type wireguard
```
<img src='https://github.com/TanunM/WireGuardVpn_HomeLab/blob/main/gallery/add_wireguard_eth.png'/>

2. Check for the new interface:
```bash
ip addr
```

### 2.2 Assign IP Address and Private Key
1. Allocate the server's private VPN IP address to the interface:
```bash
ip addr add 10.0.0.1/24 dev wg0
```

<img src='https://github.com/TanunM/WireGuardVpn_HomeLab/blob/main/gallery/wg0_ip_add.png'/>

2. Assign the generated private key to the interface:
```bash
wg set wg0 privatekey ./private
```
3. Activate the Interface
``` bash
ip link set wg0 up
```

<img src='https://github.com/TanunM/WireGuardVpn_HomeLab/blob/main/gallery/wg0_up.png'/>

## Step 3: Client Configuration and Peer Pairing

### 3.1 Install and Configure WireGuard Client (Windows)
1. Download and install the WireGuard application for Windows.
2. Click the drop-down beside "Add Tunnel" and select "Add empty tunnel"
3. The application will display a Public Key for the client. Copy this key.

<img src='https://github.com/TanunM/WireGuardVpn_HomeLab/blob/main/gallery/windows_wireguard_setup.png'/>

### 3.2 Add Client Peer to Ubuntu Server
1. On the Ubuntu Server, add the Windows client as a peer using its public key. Set its allowed internal IP to 10.0.0.2:
```bash
wg set wg0 peer <windows client public key> allowed-ips 10.0.0.2/32
```
2. In the Windows WireGuard app, start the VPN tunnel. The tunnel should immediately establish the connection.

##  Step 4: Enabling Traffic Routing
1. Enable IP Forwarding by editing the system control configuration file and saving the changes
``` bash
sudo nano /etc/sysctl.conf
{uncomment}net.ipv4.ip_forward = 1
sudo sysctl -p
```
2. Configure NAT Masquerading: Set up NAT to translate the VPN client's internal IP (10.0.0.2) to the server's public IP.
``` bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
```

<img src='https://github.com/TanunM/WireGuardVpn_HomeLab/blob/main/gallery/ip_masqurade.png'/>

3. Client Traffic Verification
   * Ensure the VPN is connected on the Windows client.
   * Disable IPv6 on the Windows client's network adapter to prevent IPv6 traffic from bypassing the VPN.

## Step 5: Ensuring Configuration Persistence Ubuntu Server
1. Save WireGuard Configuration bby exporting the current running configuration to the official persistence location:
``` bash
sudo wg showconf wg0 > /etc/wireguard/wg0.conf
```

<img src='https://github.com/TanunM/WireGuardVpn_HomeLab/blob/main/gallery/save_wg0conf.png'/>

2. Enable WireGuard System Service by enabling the service to start the wg0 interface automatically at boot:
``` bash
sudo systemctl enable wg-quick@wg0
```
3. Start the service immediately to test the persistence configuration:
``` bash
sudo systemctl start wg-quick@wg0
```
4. Save NAT Rules by installing the persistence tool for iptables rules:
``` bash
sudo apt install iptables-persistent -y
```
5. save the nat rule
``` bash
sudo netfilter-persistent save
```
6. Reboot the server
7. After the server restarts, the wg0 interface, IP forwarding, and NAT rules should be active, allowing the Windows client to immediately connect and route traffic.

## Troubleshooting
* No Internet after VPN connection.: Check IP Forwarding: Ensure net.ipv4.ip_forward = 1 is enabled in /etc/sysctl.conf and loaded with sysctl -p
* Connection fails after server reboot: Persistence failed: Verify the configuration is saved to /etc/wireguard/wg0.conf and that the systemd service is enabled: sudo systemctl status wg-quick@wg0. Check that netfilter-persistent save was run.
* Client traffic isn't routed (bypassing VPN): Ensure the Windows client's endpoint configuration is correct and that IPv6 is disabled on the client's physical adapter.

## Key Learnings
* Public Key Cryptography: WireGuard uses a public/private key pair to establish a secure, authenticated tunnel, which is faster and more secure than traditional password/certificate methods.
* Layer 3 Tunneling: WireGuard operates at Layer 3 (IP layer), efficiently creating a virtual network interface (wg0).
* NAT (Masquerading): Essential for a VPN server to act as a gateway, allowing clients with private IP addresses to access the public internet using the server's public IP.
* Linux Networking Primitives: The lab utilizes core Linux tools (ip, iptables, sysctl) to configure the network stack, demonstrating fundamental networking administration skills.
* Systemd Persistence: Proper use of the wg-quick@ systemd unit and iptables-persistent is crucial for production environments, ensuring the VPN service is reliable after system events like reboots.

## Reference
* [Host Your Own WireGuard](https://www.youtube.com/watch?v=O2mxQSqvsaM)
