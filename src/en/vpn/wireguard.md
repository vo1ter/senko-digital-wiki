---
title: "WireGuard VPN Installation Guide"
description: "Complete guide for installing and configuring a WireGuard VPN server on Linux distributions including Debian, Ubuntu, CentOS, Fedora, and AlmaLinux"
head:
  - - meta
    - name: keywords
      content: wireguard, vpn, server, installation, debian, ubuntu, centos, fedora, almalinux, guide, configuration
  - - meta
    - property: og:title
      content: "WireGuard VPN - Complete Installation Guide"
  - - meta
    - property: og:description
      content: "Step-by-step guide for installing and configuring a WireGuard VPN server on various Linux distributions."
---

# WireGuard VPN Installation Guide

<img src="/images/vpn/wireguard-easy/wg-easy-logo.png" style="display: flex !important; justify-content: center !important; height: 500px;"></img>

WireGuard is a modern, fast, and secure VPN protocol that's easy to configure and deploy. This guide will walk you through installing and configuring WireGuard on your server to create a secure VPN connection.

::: tip
Our hosting clients have access to a fully automated WireGuard installation script during initial service ordering, or through scripts in the VM control panel.
:::

## Prerequisites

Before starting, ensure you have:

- A server running a supported Linux distribution (Debian/Ubuntu, RHEL/CentOS/AlmaLinux/Rocky/Fedora)
- Root or sudo access to the server
- Basic knowledge of command line operations
- The server's public IP address

## Installation

### Automatic Installation for Hosting Clients

#### During Service Order

In the order menu, select "WireGuard" from the dropdown menu, complete the order and wait for service processing to finish.

Immediately after server activation, you will receive an email containing your WireGuard configuration file and instructions for connecting. The configuration file can be imported directly into any WireGuard client application.

#### Through the VM Panel

Go to the [VM control panel](https://vm.senko.digital), navigate to the management of the desired server, expand the "Menu" in the top right corner and select "Run script".

In the dialog, select the "WireGuard" option and enable "Send email associated with script".

After installation is complete, you will receive an email with your WireGuard configuration file and connection instructions. Simply import the configuration file into your WireGuard client to start using the VPN.

#### What Gets Installed

The script deploys a WireGuard VPN server inside Docker with automatic client configuration generation.

After installation you receive:

- **peer1.conf** — a ready-to-use WireGuard client configuration file (also displayed as a QR code in the script log)

#### How to Connect

1. Install the [WireGuard client](https://www.wireguard.com/install/) on your device
2. Import the `peer1.conf` file (or scan the QR code on mobile)
3. Activate the tunnel

#### Firewall

The script configures a **restrictive** firewall — only SSH and WireGuard (`51820/udp`) are open. All other inbound traffic is dropped.

#### Server Management

```bash
cd /opt/wireguard-docker && docker compose up -d    # start
cd /opt/wireguard-docker && docker compose down      # stop
cd /opt/wireguard-docker && docker compose logs -f   # logs
```

### Manual Installation

The installation process varies depending on your Linux distribution. Make sure to use the correct commands for the installation process.

::: code-group

```bash [Ubuntu/Debian]
# Update package list
apt-get update -y

# Install WireGuard and required tools
apt-get install -y wireguard wireguard-tools iproute2 iptables
```

```bash [RHEL/CentOS/AlmaLinux/Rocky/Fedora]
# Install EPEL repository (if not already installed)
yum install -y epel-release
# OR for newer systems:
dnf install -y epel-release

# Install WireGuard and required tools
yum install -y wireguard-tools iproute iptables
# OR for newer systems:
dnf install -y wireguard-tools iproute iptables
```

:::

## Server Configuration

### 1. Create Directory Structure

First, create the necessary directories for storing WireGuard configuration files:

```bash
# Set secure permissions
umask 077

# Create directories for server and client configurations
mkdir -p /etc/wireguard/server
mkdir -p /etc/wireguard/client
```

### 2. Generate Cryptographic Keys

WireGuard uses public-key cryptography. Generate key pairs for both server and client:

```bash
# Generate server keys
wg genkey | tee /etc/wireguard/server/privatekey | wg pubkey > /etc/wireguard/server/publickey

# Generate client keys
wg genkey | tee /etc/wireguard/client/privatekey | wg pubkey > /etc/wireguard/client/publickey
```

### 3. Plan Network Parameters

Identify your default network interface and choose addressing parameters:

- **Network interface**: Find it with `ip route get 1.1.1.1 | awk '{print $5; exit}'` or inspect `ip route` (commonly `eth0` or `ens3`)
- **Server public IP**: Confirm via `curl -4 ifconfig.io` or in the Client Area
- **VPN subnet**: Choose a private subnet that does not conflict with your existing networks, e.g., `10.8.0.0/24` (use `10.9.0.0/24` or another RFC1918 range if `10.8.0.0/24` conflicts)
- **Server VPN address**: Use the first usable IP in the chosen subnet, e.g., `10.8.0.1/24`
- **Client VPN address**: Assign a unique IP per client, e.g., `10.8.0.2/24`
- **Port**: `51820/udp` by default (you may choose a different UDP port)

### 4. Enable IP Forwarding

Enable IP forwarding to allow the server to route traffic between the VPN and the internet:

```bash
# Enable IP forwarding temporarily
sysctl -w net.ipv4.ip_forward=1
sysctl -w net.ipv6.conf.all.forwarding=1

# Apply changes
sysctl -p

# Make the change permanent
echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-wireguard-forwarding.conf
echo 'net.ipv6.conf.all.forwarding=1' >> /etc/sysctl.d/99-wireguard-forwarding.conf
```

### 5. Create Server Configuration

Create the main WireGuard configuration file at `/etc/wireguard/wg0.conf`:

```ini
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = [SERVER_PRIVATE_KEY]
PostUp = iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o [INTERFACE] -j MASQUERADE; iptables -A FORWARD -i %i -o [INTERFACE] -j ACCEPT; iptables -A FORWARD -i [INTERFACE] -o %i -m state --state RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.8.0.0/24 -o [INTERFACE] -j MASQUERADE; iptables -D FORWARD -i %i -o [INTERFACE] -j ACCEPT; iptables -D FORWARD -i [INTERFACE] -o %i -m state --state RELATED,ESTABLISHED -j ACCEPT

[Peer]
PublicKey = [CLIENT_PUBLIC_KEY]
AllowedIPs = 10.8.0.2/32
```

**Important**: Replace the placeholders:

- `[SERVER_PRIVATE_KEY]`: Content of `/etc/wireguard/server/privatekey`
- `[CLIENT_PUBLIC_KEY]`: Content of `/etc/wireguard/client/publickey`
- `[INTERFACE]`: Your server's network interface (e.g., `eth0`, `ens3`)

Set secure permissions on the configuration file:

```bash
chmod 600 /etc/wireguard/wg0.conf
```

## Client Configuration

Create a client configuration file at `/etc/wireguard/client/client.conf`:

```ini
[Interface]
PrivateKey = [CLIENT_PRIVATE_KEY]
Address = 10.8.0.2/24
DNS = 1.1.1.1, 1.0.0.1

[Peer]
PublicKey = [SERVER_PUBLIC_KEY]
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = [SERVER_IP]:51820
PersistentKeepalive = 25
```

**Replace the placeholders**:

- `[CLIENT_PRIVATE_KEY]`: Content of `/etc/wireguard/client/privatekey`
- `[SERVER_PUBLIC_KEY]`: Content of `/etc/wireguard/server/publickey`
- `[SERVER_IP]`: Your server's public IP address

Set secure permissions:

```bash
chmod 600 /etc/wireguard/client/client.conf
```

## Starting the VPN

### Enable and Start WireGuard Service

```bash
# Enable WireGuard to start automatically on boot
systemctl enable wg-quick@wg0

# Start the WireGuard service
systemctl start wg-quick@wg0
```

Alternative method using `wg-quick`:

```bash
# Start WireGuard interface
wg-quick up wg0

# Stop WireGuard interface
wg-quick down wg0
```

## Verification

### Check WireGuard Status

```bash
# Check if WireGuard is running
systemctl status wg-quick@wg0

# View WireGuard interface details
wg show

# Check network interfaces
ip addr show wg0
```

### Test the VPN Connection

1. **Copy the client configuration** to your client device
2. **Install WireGuard client** on your device
3. **Import the configuration** file
4. **Connect to the VPN**
5. **Verify your IP address** has changed to your server's IP

## Firewall Configuration

If you're using a firewall (like `ufw` or `firewalld`), make sure to allow WireGuard traffic:

### For UFW (Ubuntu/Debian):

```bash
# Allow WireGuard port
ufw allow 51820/udp

# Allow forwarding (if needed)
ufw route allow in on wg0 out on eth0
```

### For firewalld (RHEL/CentOS):

```bash
# Allow WireGuard port
firewall-cmd --permanent --add-port=51820/udp
firewall-cmd --reload

# Enable masquerading
firewall-cmd --permanent --add-masquerade
firewall-cmd --reload
```

## Adding Additional Clients

To add more clients to your WireGuard server:

1. **Create client directory**:

   ```bash
   mkdir -p /etc/wireguard/client2
   chmod 700 /etc/wireguard/client2
   ```

2. **Generate a new key pair** for the new client:

   ```bash
   wg genkey | tee /etc/wireguard/client2/privatekey | wg pubkey > /etc/wireguard/client2/publickey
   ```

3. **Add new peer section** to `/etc/wireguard/wg0.conf`:

   ```ini
   [Peer]
   PublicKey = [CLIENT2_PUBLIC_KEY]
   AllowedIPs = 10.8.0.3/32
   ```

4. **Reload WireGuard configuration**:
   ```bash
   systemctl reload wg-quick@wg0
   ```

## Installing WireGuard Client

### Windows Installation

1. Go to the official website [wireguard.com](https://www.wireguard.com/install/) and download the Windows installer

![Downloading WireGuard client for Windows](/images/vpn/wireguard-easy/windows-download-wireguard.png){data-zoomable}

2. Run the downloaded file and follow the installation wizard

![Running WireGuard installer](/images/vpn/wireguard-easy/windows-open-installer.png){data-zoomable}

![WireGuard installation process](/images/vpn/wireguard-easy/windows-install-wireguard.png){data-zoomable}

The application will launch automatically after installation.

3. Import your configuration file into the application

<video width="1440" autoplay controls loop muted>
  <source src="/videos/vpn/wireguard-easy/import-config-windows.mp4" type="video/mp4" />
</video>

4. Start the VPN connection

Click the "Activate" button.

![Activating VPN connection in WireGuard on Windows](/images/vpn/wireguard-easy/windows-connect.png){data-zoomable}

**Congratulations!**

All your network traffic is now protected and routed through the VPN tunnel.

### macOS Installation

1. Find and install the WireGuard client through the Mac App Store

![WireGuard page in Mac App Store](/images/vpn/wireguard-easy/app-store.png){data-zoomable}

2. Find and open the installed application

![Searching for WireGuard in Spotlight on macOS](/images/vpn/wireguard-easy/macos-wireguard-search.png){data-zoomable}

3. Add your configuration to the program

<video width="1440" autoplay controls loop muted>
  <source src="/videos/vpn/wireguard-easy/import-config-macos.mp4" type="video/mp4" />
</video>

On first launch, macOS will request permission to create VPN connections.

**Make sure to click "Allow", otherwise the application won't function properly**.

Activate the VPN connection.

Click the "Activate" button.

![Enabling VPN connection in WireGuard on macOS](/images/vpn/wireguard-easy/macos-connect.png){data-zoomable}

**Great!**

Your internet connection is now protected by the VPN tunnel.

### Mobile Devices (Android/iOS)

1. Download the official WireGuard app:
   - For Android: [Google Play Store](https://play.google.com/store/apps/details?id=com.wireguard.android&hl=en)
   - For iOS: [App Store](https://apps.apple.com/us/app/wireguard/id1441195209)
2. Import the configuration file
3. Activate the VPN connection in the app

## Troubleshooting

### Common Issues

1. **Connection fails**:

   - Check if port 51820/udp is open in your firewall
   - Verify the server's public IP address is correct
   - Ensure IP forwarding is enabled

2. **No internet access through VPN**:

   - Check iptables rules are applied correctly
   - Verify the network interface name in the configuration
   - Ensure masquerading is enabled
   - Make sure there are no restrictions applied for the protocol on your Internet Service Provider's side

3. **WireGuard service won't start**:
   - Check configuration file syntax: `wg-quick up wg0`
   - Verify file permissions are set correctly (600)
   - Check system logs: `journalctl -u wg-quick@wg0`

### Useful Commands

```bash
# View detailed WireGuard status
wg show all

# Check WireGuard logs
journalctl -u wg-quick@wg0 -f

# Test configuration without starting
wg-quick up wg0 --dry-run

# Restart WireGuard
systemctl restart wg-quick@wg0
```

### Network Debugging

```bash
# Check routing table
ip route show table all

# Monitor network traffic
tcpdump -i wg0

# Test connectivity
ping -I wg0 8.8.8.8
```

## Security Considerations

1. **Keep private keys secure**: Never share private keys and ensure they have proper file permissions (600)
2. **Regular key rotation**: Consider rotating keys periodically for enhanced security
3. **Firewall rules**: Only allow necessary traffic through your firewall
4. **Monitor connections**: Regularly check connected clients with `wg show`
5. **Update regularly**: Keep WireGuard and your system updated

## Conclusion

You now have a fully functional WireGuard VPN server! The client configuration file can be imported into any WireGuard client application on various devices (Windows, macOS, iOS, Android, Linux) to establish a secure VPN connection to your server.

Remember to securely transfer the client configuration to your devices and keep your private keys safe.
