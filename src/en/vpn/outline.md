---
title: "Outline VPN Installation Guide"
description: "Complete guide for installing and configuring an Outline VPN server on Linux distributions including Debian, Ubuntu, CentOS, Fedora, and AlmaLinux"
head:
  - - meta
    - name: keywords
      content: outline, vpn, server, installation, debian, ubuntu, centos, fedora, almalinux, guide, configuration, shadowsocks
  - - meta
    - property: og:title
      content: "Outline VPN - Complete Installation Guide"
  - - meta
    - property: og:description
      content: "Step-by-step guide for installing and configuring an Outline VPN server on various Linux distributions."
---

# Outline VPN Installation Guide

<img src="/images/vpn/outline/outline-logo.png" style="display: flex !important; justify-content: center !important; height: 500px;"></img>

Outline is a modern, fast, and secure VPN solution developed by Jigsaw (a subsidiary of Google). It uses the Shadowsocks protocol and provides an easy-to-use management app for configuring your VPN server and clients. This guide will walk you through installing and configuring Outline on your server.

:::: tip
Our hosting clients have access to a fully automated Outline installation script during initial service ordering, or through scripts in the VM control panel. After the script finishes, you'll receive connection details for Outline Connect and management details for Outline Manager.
::::

## Prerequisites

Before starting, ensure you have:

- A server running a supported Linux distribution (Debian/Ubuntu, RHEL/CentOS/AlmaLinux/Rocky/Fedora)
- Root or sudo access to the server
- Basic knowledge of command line operations
- The server's public IP address
- A device to install the Outline Manager app (Windows, macOS, or Linux)

## Installation

### Automatic Installation for Hosting Clients

#### During Service Order

In the order menu, select "Outline" from the dropdown menu, complete the order and wait for service processing to finish.

Immediately after server activation, you will receive an email containing your Outline Manager URL and access credentials. You can use these to connect to your server through the Outline client, or manage your VPN server through the Outline Manager app.

#### Through the VM Panel

Go to the [VM control panel](https://vm.senko.digital), navigate to the management of the desired server, expand the "Menu" in the top right corner and select "Run script".

In the dialog, select the "Outline" option and enable "Send email associated with script".

After installation is complete, you will receive an email with your Outline Manager URL and access credentials. You can then use these to connect to your server through the Outline client, or manage your VPN server through the Outline Manager app.

#### What Gets Installed

The script deploys an Outline VPN server (Shadowsocks-based) inside Docker, managed via the Outline Manager desktop app.

After installation you receive:

- **connection-info.txt** — contains the Management API URL and certificate fingerprint needed for Outline Manager

#### How to Connect

1. Install [Outline Manager](https://getoutline.org/) on your PC/Mac
2. Choose "Set up Outline anywhere" and paste the management URL from `connection-info.txt`
3. Create access keys in Outline Manager and share them with users
4. Users install the [Outline Client](https://getoutline.org/) and paste their access key

#### Firewall

The script configures an **open** firewall — no inbound restrictions are applied (Outline requires dynamic ports for client connections).

#### Server Management

```bash
cd /opt/outline-docker && docker compose up -d    # start
cd /opt/outline-docker && docker compose down      # stop
cd /opt/outline-docker && docker compose logs -f   # logs
```

### Manual Installation

The installation process varies depending on your Linux distribution. Make sure to use the correct commands for the installation process.

::: code-group

```bash [Ubuntu/Debian]
# Update package list
apt-get update -y

# Install required dependencies
apt-get install -y curl jq
```

```bash [RHEL/CentOS/AlmaLinux/Rocky/Fedora]
# Install EPEL repository (if not already installed)
yum install -y epel-release
# OR for newer systems:
dnf install -y epel-release

# Install required dependencies
yum install -y curl jq
# OR for newer systems:
dnf install -y curl jq
```

:::

#### Run the Official Outline Installer

The Outline project provides an official installation script that handles all the complex setup:

```bash
# Download and run the official Outline installer
curl -sS https://raw.githubusercontent.com/Jigsaw-Code/outline-server/master/src/server_manager/install_scripts/install_server.sh | bash
```

This script will:
- Install Docker and Docker Compose
- Set up the Outline server with Shadowsocks
- Configure the management interface
- Generate server certificates
- Start all necessary services

## Installing the Outline Manager App

After installation, you'll need to install the Outline Manager app to configure your VPN server and create client access keys.

### Download the Outline Manager App

Download the Outline Manager app for your platform:

- **Windows**: [Download from official website](https://getoutline.org/get-started/#step-1)
- **macOS**: [Download from official website](https://getoutline.org/get-started/#step-1) or [Mac App Store](https://apps.apple.com/app/outline-manager/id1356177745)
- **Linux**: [Download from official website](https://getoutline.org/get-started/#step-1)

### Finding Your Server Configuration

The installation script will output a JSON configuration containing your server details. Look for output similar to:

```json
{
  "apiUrl": "https://your-server-ip:port/your-server-id",
  "certSha256": "your-certificate-hash"
}
```

Copy these details for later.

If you can't find the server configuration, you can check the installation logs:

```bash
# Check the installation log for the server configuration
cat /var/log/outline-install.log | grep -E "(apiUrl|certSha256)"
```

### Adding Your Server to Outline Manager

1. **Install and launch** the Outline Manager app
2. **Choose "Set up Outline anywhere"** in the provider selection
![Outline Manager window with server provider selection and an arrow pointing towards "Set up Outline anywhere"](/images/vpn/outline/set-up-anywhere.png){data-zoomable}
3. **Paste your server configuration** (the JSON containing `apiUrl` and `certSha256`) copied in the previous step
![Outline Manager window with a prompt to paste server configuration](/images/vpn/outline/paste-credentials.png){data-zoomable}
4. **Set a server name** (optional but recommended)
5. **Click "Done"** to add your server

After successful setup, the app will take you straight to your server dashboard:

![Outline Manager dashboard](/images/vpn/outline/dashboard.png){data-zoomable}

## Server Configuration

### Initial Server Setup

Once your server is added to the Outline Manager app, you can configure it:

1. **Set a server name** - Choose a descriptive name for your VPN server
2. **Set data limits** - Optionally set monthly data limits for users

![Outline Manager server settings](/images/vpn/outline/settings.png){data-zoomable}

### Creating Access Keys

Access keys are what clients use to connect to your Outline server. Each key represents a unique connection.

#### Creating Your First Access Key

1. **Click "Add new key"** in the Outline Manager app
![Outline Manager - Add new key](/images/vpn/outline/add-new-key.png){data-zoomable}
2. **Choose a name** for the key (e.g., "My Phone", "Laptop")
3. **Set data limits** (optional)
4. **Click "Add"** to create the key

The system will immediately generate a unique access URL (starting with `ss://`) that clients will use to connect.

With Outline Manager you can:

- **Rename keys**
- **Monitor traffic usage** per key
- **Set or reset data limits**

To view/share a key, click the **Share** icon next to it, and the access key will be shown.

![Outline Manager - Access key](/images/vpn/outline/access-key.png){data-zoomable}

Provide it to your users, or use it in the next step.

## Client Configuration

### Installing Outline Connect

Outline provides official client applications called "Outline Connect" for various platforms.

#### Windows Installation

1. **Download Outline Connect** from the [official website](https://getoutline.org/get-started/#step-3) or the [direct mirror](https://s3.amazonaws.com/outline-releases/client/windows/stable/Outline-Client.exe)
2. **Run the installer** and follow the setup wizard
3. **Launch Outline Connect** after installation

#### macOS Installation

1. **Download Outline Connect** from the [Mac App Store](https://apps.apple.com/app/outline-app/id1356178125) or the [official website](https://getoutline.org/get-started/#step-3)
2. **Install the application** following the standard macOS installation process
3. **Launch Outline Connect** from your Applications folder

#### Linux Installation

1. **Download the AppImage** from the [official mirror](https://s3.amazonaws.com/outline-releases/client/linux/stable/Outline-Client.AppImage)
2. Make it executable: `chmod +x Outline-Client.AppImage`
3. Run the application

#### Mobile Devices (Android/iOS)

1. **Download the app**:
   - For Android: [Google Play Store](https://play.google.com/store/apps/details?id=org.outline.android.client) or [direct APK](https://s3.amazonaws.com/outline-releases/client/android/stable/Outline-Client.apk)
   - For iOS: [App Store](https://apps.apple.com/app/outline-app/id1356178125)
2. **Install and launch** the Outline Connect app

### Connecting to Your VPN

1. **Open Outline Connect** on your device
2. **Add a server** by clicking the "+" button
3. **Enter the access key** (the `ss://` URL you copied from the Outline Manager)
4. **Give the connection a name** (optional)
5. **Click "Add"** to save the connection
6. **Connect** by clicking the toggle switch next to your server

## Verification

### Check Server Status

You can verify that your Outline server is running properly:

```bash
# Check if Docker containers are running
docker ps | grep outline

# Check Outline server logs
docker logs outline-server

# Check if the management interface is accessible
curl -k https://localhost:your-port/your-server-id
```

### Test the VPN Connection

1. **Connect to your VPN** using Outline Connect
2. **Check your IP address** - Visit [check-host.net](https://check-host.net) to verify your IP has changed
3. **Test internet connectivity** - Ensure you can browse the web normally
4. **Check connection stability** - Leave the connection active for a while to test stability

## Firewall Configuration

Outline uses Shadowsocks protocol and typically runs on a custom port. Make sure your firewall allows the necessary traffic:

### For UFW (Ubuntu/Debian):

```bash
# Allow the Outline port (check the actual port in your manager URL)
ufw allow 443/tcp
ufw allow 80/tcp

# If using a custom port, replace 443 with your actual port
ufw allow CUSTOM/tcp
```

### For firewalld (RHEL/CentOS):

```bash
# Allow the Outline port
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --reload

# If using a custom port, replace 443 with your actual port
firewall-cmd --permanent --add-port=CUSTOM/tcp
firewall-cmd --reload
```

## Managing Users and Access

### Adding Multiple Users

1. **Create additional access keys** in the Outline Manager app
2. **Share the access URLs** with your users
3. **Set individual data limits** for each user if needed
4. **Monitor usage** through the management app

### User Management Features

- **Usage statistics** - View data usage for each access key
- **Connection monitoring** - See which keys are currently connected
- **Data limits** - Set and enforce monthly data limits
- **Key management** - Rename, modify, or delete access keys

## Troubleshooting

### Common Issues

1. **Cannot connect to server in Outline Manager**:
   - Check if the server is running: `docker ps | grep outline`
   - Verify the server configuration (apiUrl and certSha256) is correct
   - Ensure the port is open in your firewall
   - Check if the server is accessible from your network

2. **Client cannot connect**:
   - Verify the access key URL is correct
   - Check if the server port is accessible
   - Ensure the client app is up to date
   - Try regenerating the access key

3. **Slow connection speeds**:
   - Check server resources (CPU, memory, bandwidth)
   - Verify your internet connection speed
   - Try connecting from a different network
   - Check if there are any bandwidth limits set

4. **Outline Manager app shows errors**:
   - Check Docker container logs: `docker logs outline-server`
   - Restart the Outline service: `docker restart outline-server`
   - Verify disk space and system resources
   - Try removing and re-adding the server in the Outline Manager app

### Useful Commands

```bash
# Check Outline server status
docker ps | grep outline

# View server logs
docker logs outline-server

# Restart Outline server
docker restart outline-server

# Check system resources
htop
df -h

# Test network connectivity
ping 8.8.8.8
curl -I https://www.google.com
```

### Network Debugging

```bash
# Check if the Outline port is listening
netstat -tlnp | grep :your-port

# Test port connectivity
telnet your-server-ip your-port

# Check firewall status
ufw status
# OR
firewall-cmd --list-all
```

## Security Considerations

1. **Keep access keys secure**: Never share access keys publicly and rotate them regularly
2. **Use strong passwords**: Set a strong password for the Outline Manager
3. **Regular updates**: Keep your server and Outline software updated
4. **Monitor usage**: Regularly check the management interface for unusual activity
5. **Backup configuration**: Keep backups of your server configuration
6. **Network security**: Ensure your server is properly secured and firewalled

## Advanced Configuration

### Custom Port Configuration

If you need to change the default port:

1. **Stop the Outline server**: `docker stop outline-server`
2. **Edit the configuration** in the Docker compose file
3. **Restart the server**: `docker start outline-server`
4. **Update firewall rules** to allow the new port

### SSL Certificate Configuration

For production use, consider replacing the self-signed certificate with a proper SSL certificate:

1. **Obtain an SSL certificate** (Let's Encrypt recommended)
2. **Update the Outline configuration** to use the new certificate
3. **Restart the Outline server**

## Conclusion

You now have a fully functional Outline VPN server! The Outline Manager app makes it easy to manage users, monitor usage, and configure your VPN server. Users can connect using the official Outline Connect applications on any platform.

Remember to:
- Keep your access keys secure
- Regularly monitor usage and performance
- Keep your server and software updated
- Back up your configuration regularly

The Outline Manager app provides an intuitive interface for all server management tasks, making it easy to maintain and scale your VPN service.
