Sure — here is the same runbook as a clean Markdown document you can save as `ubuntu-server-network-setup.md`.

# Ubuntu Server — Network & SSH Setup

This document records the steps used to configure Internet access, Wi-Fi, SSH, and a DHCP reservation for the Ubuntu Server.

---

## 1. Identify Network Interfaces

Check the available network interfaces:

```bash
ip addr
```

The server had three interfaces:

| Interface         | Purpose  |
| ----------------- | -------- |
| `lo`              | Loopback |
| `enp5s0`          | Ethernet |
| `wlx9848277f0c8a` | Wi-Fi    |

Since the server is using Wi-Fi, the relevant interface is:

```text
wlx9848277f0c8a
```

---

## 2. Enable the Wi-Fi Interface

Initially, the Wi-Fi interface showed:

```text
NO-CARRIER
state DOWN
```

Bring the interface up:

```bash
sudo ip link set wlx9848277f0c8a up
```

Note that this only enables the Wi-Fi adapter. It does **not** connect the adapter to a Wi-Fi network.

---

## 3. Configure Wi-Fi Using Netplan

`nmcli` and `iw` were not installed on the minimal Ubuntu Server installation.

However, `wpa_supplicant` was available:

```bash
which wpa_supplicant
```

Output:

```text
/usr/sbin/wpa_supplicant
```

The `/etc/netplan/` directory was initially empty:

```bash
ls -l /etc/netplan/
```

Output:

```text
total 0
```

Create a Netplan configuration:

```bash
sudo nano /etc/netplan/01-wifi.yaml
```

Add:

```yaml
network:
  version: 2
  wifis:
    wlx9848277f0c8a:
      dhcp4: true
      access-points:
        "YOUR_WIFI_NAME":
          password: "YOUR_WIFI_PASSWORD"
```

Replace:

```text
YOUR_WIFI_NAME
```

with the Wi-Fi SSID and:

```text
YOUR_WIFI_PASSWORD
```

with the Wi-Fi password.

> **Do not share the Wi-Fi password in chat or documentation.**

---

## 4. Secure the Netplan Configuration

Netplan warned that the configuration file was too open because it contains the Wi-Fi password.

Restrict access to the file:

```bash
sudo chmod 600 /etc/netplan/01-wifi.yaml
```

Verify:

```bash
ls -l /etc/netplan/01-wifi.yaml
```

Expected permissions:

```text
-rw------- 1 root root ... 01-wifi.yaml
```

This means only `root` can read or modify the file.

---

## 5. Validate and Apply Netplan

First validate/generate the configuration:

```bash
sudo netplan generate
```

If there are no errors, apply it:

```bash
sudo netplan apply
```

Check the Wi-Fi interface:

```bash
ip addr show wlx9848277f0c8a
```

After successful connection, the interface received:

```text
192.168.1.18/24
```

Therefore, the Ubuntu Server's current IP address is:

```text
192.168.1.18
```

---

## 6. Test Internet Connectivity

Test Internet connectivity directly:

```bash
ping -c 4 8.8.8.8
```

Then test DNS resolution:

```bash
ping -c 4 archive.ubuntu.com
```

The two tests check different things:

```text
8.8.8.8 works
archive.ubuntu.com fails
        ↓
DNS problem
```

Whereas:

```text
8.8.8.8 works
archive.ubuntu.com works
        ↓
Internet + DNS working
```

---

# SSH Configuration

## 7. Install OpenSSH Server

Update the package repository:

```bash
sudo apt update
```

Install OpenSSH Server:

```bash
sudo apt install openssh-server -y
```

---

## 8. Enable and Start SSH

Check the SSH service:

```bash
sudo systemctl status ssh
```

If SSH is disabled/not running, enable and start it:

```bash
sudo systemctl enable --now ssh
```

This performs two actions:

- `enable` — starts SSH automatically when Ubuntu boots.
- `--now` — starts SSH immediately.

Verify:

```bash
sudo systemctl status ssh
```

Expected status:

```text
Active: active (running)
```

and:

```text
enabled
```

---

## 9. Connect to Ubuntu from Windows

The Ubuntu Server currently has:

```text
192.168.1.18
```

From Windows PowerShell:

```powershell
ssh YOUR_USERNAME@192.168.1.18
```

For example:

```powershell
ssh anurag@192.168.1.18
```

The first connection may display:

```text
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

Enter:

```text
yes
```

Then enter the Ubuntu user's password.

After this, Ubuntu can be administered remotely from Windows without requiring a monitor and keyboard.

---

# DHCP Reservation

## 10. Why Use a DHCP Reservation?

The current Netplan configuration uses:

```yaml
dhcp4: true
```

This means the router assigns an IP address to Ubuntu.

Currently:

```text
Ubuntu Server
     |
     | DHCP request
     ↓
  Router
     |
     ↓
192.168.1.18
```

The router could potentially assign a different IP after a reboot or lease expiration.

For example:

```text
192.168.1.18
```

could later become:

```text
192.168.1.25
```

Then this command would no longer work:

```powershell
ssh anurag@192.168.1.18
```

A DHCP reservation tells the router:

> Whenever this particular network adapter connects, assign it the same IP address.

---

## 11. Get the Wi-Fi MAC Address

Run:

```bash
ip link show wlx9848277f0c8a
```

Look for:

```text
link/ether XX:XX:XX:XX:XX:XX
```

Alternatively:

```bash
cat /sys/class/net/wlx9848277f0c8a/address
```

Example:

```text
98:48:27:7f:0c:8a
```

The actual MAC address on your system should be used when configuring the router.

---

# 12. Find the Router's IP Address

From Windows PowerShell:

```powershell
ipconfig
```

Look for:

```text
Default Gateway . . . . . : 192.168.1.1
```

The gateway may be different depending on your router.

Open the gateway address in a browser:

```text
http://192.168.1.1
```

---

# 13. Find DHCP Settings on the Router

The exact menu depends on the router manufacturer.

Look for settings such as:

- DHCP Reservation
- Address Reservation
- Static DHCP
- Reserved IP
- DHCP Binding
- IP & MAC Binding

Common menu locations include:

```text
Network
  └── LAN
      └── DHCP Server
```

or:

```text
Advanced
  └── LAN Setup
      └── DHCP
```

---

# 14. Create the DHCP Reservation

Create a new reservation using:

```text
Device/MAC Address:
<Ubuntu Wi-Fi MAC address>

Reserved IP:
192.168.1.18

Description:
Ubuntu Server
```

For example:

```text
MAC Address: 98:48:27:7f:0c:8a
IP Address:  192.168.1.18
Name:        Ubuntu Server
```

The exact interface will depend on your router.

Save/apply the configuration.

---

# 15. Verify the DHCP Reservation

On Ubuntu:

```bash
ip addr show wlx9848277f0c8a
```

The server should receive:

```text
192.168.1.18
```

Check the routing table:

```bash
ip route
```

You should see something similar to:

```text
default via 192.168.1.1
```

From Windows, test SSH:

```powershell
ssh anurag@192.168.1.18
```

---

# Final Network Architecture

```text
                         Internet
                            |
                            |
                     +------+------+
                     |   Router    |
                     |             |
                     | DHCP Server |
                     +------+------+
                            |
                       Wi-Fi Network
                            |
                     192.168.1.x
                            |
                            v
                 +--------------------+
                 |   Ubuntu Server    |
                 |                    |
                 | Wi-Fi Adapter      |
                 | 192.168.1.18       |
                 |                    |
                 | Netplan            |
                 |       |            |
                 |       v            |
                 | wpa_supplicant     |
                 |                    |
                 | OpenSSH Server     |
                 +---------+----------+
                           |
                           | SSH
                           |
                           v
                 +--------------------+
                 |     Windows 10     |
                 |                    |
                 | ssh user@           |
                 | 192.168.1.18       |
                 +--------------------+
```

---

# Important Commands

### Network

```bash
ip addr
```

```bash
ip addr show wlx9848277f0c8a
```

```bash
ip route
```

### Wi-Fi

```bash
sudo ip link set wlx9848277f0c8a up
```

### Netplan

```bash
sudo netplan generate
```

```bash
sudo netplan apply
```

### Test Internet

```bash
ping -c 4 8.8.8.8
```

```bash
ping -c 4 archive.ubuntu.com
```

### Update Ubuntu

```bash
sudo apt update
```

### SSH

```bash
sudo apt install openssh-server -y
```

```bash
sudo systemctl enable --now ssh
```

```bash
sudo systemctl status ssh
```

### MAC Address

```bash
cat /sys/class/net/wlx9848277f0c8a/address
```

### SSH from Windows

```powershell
ssh YOUR_USERNAME@192.168.1.18
```

---

## Current Server Configuration

| Item                | Value                       |
| ------------------- | --------------------------- |
| OS                  | Ubuntu Server               |
| Network             | Wi-Fi                       |
| Wi-Fi interface     | `wlx9848277f0c8a`           |
| Ethernet interface  | `enp5s0`                    |
| Wi-Fi configuration | Netplan                     |
| IP assignment       | DHCP                        |
| Current IP          | `192.168.1.18`              |
| SSH                 | OpenSSH Server              |
| SSH startup         | Enabled                     |
| SSH access          | `user@192.168.1.18`         |
| DHCP reservation    | To be configured on router  |
| Netplan config      | `/etc/netplan/01-wifi.yaml` |
| Netplan permissions | `600`                       |
