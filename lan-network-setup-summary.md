# Ubuntu Server — Ethernet Network & Static IP Setup

## Session Summary

**Date:** October 4, 2026

The Ubuntu Server was previously connected using a Wi-Fi dongle. The Wi-Fi dongle was removed and the server was connected directly to the ISP router using Ethernet.

The Ethernet interface is `enp5s0`.

## 1. Initial Ethernet State

After connecting Ethernet, `ip addr` showed `enp5s0` as `DOWN`. The interface was detected but was not active.

It was brought up with:

```bash
sudo ip link set enp5s0 up
```

The existing Netplan configuration had previously been for Wi-Fi, so it was changed to configure Ethernet.

## 2. DHCP Connectivity Test

The Ethernet interface was initially configured with DHCP:

```yaml
network:
  version: 2
  ethernets:
    enp5s0:
      dhcp4: true
```

The server successfully received:

```text
IP address: 192.168.1.28/24
Gateway:    192.168.1.1
Interface:  enp5s0
```

`dhclient` was not installed. This was not a problem because the Ubuntu Server setup uses Netplan/systemd-networkd for network configuration.

## 3. Internet Connectivity Verification

All connectivity tests succeeded.

### Router

```bash
ping -c 4 192.168.1.1
```

Result: **Success**

### Internet without DNS

```bash
ping -c 4 8.8.8.8
```

Result: **Success**

### DNS and Internet

```bash
ping -c 4 google.com
```

Result: **Success**

Therefore Ethernet, LAN routing, Internet access, and DNS were all confirmed working.

## 4. Static IP Decision

The server was assigned the static IP:

```text
192.168.1.200
```

The ISP router does not currently provide user-controlled DHCP reservations. Its DHCP range was previously observed as:

```text
DHCP start: 192.168.1.2
DHCP end:   192.168.1.254
```

The router also has a maximum connection setting of 64. The current assumption is that DHCP allocation is sequential, but this has not been formally confirmed with the ISP.

Therefore `192.168.1.200` is being used as a practical temporary static address. There is a small risk that the ISP router could eventually allocate this address to another device.

## 5. Final Netplan Configuration

The active configuration file is:

```text
/etc/netplan/03-lan-static-ip.yaml
```

Current configuration:

```yaml
network:
  version: 2
  ethernets:
    enp5s0:
      addresses:
        - 192.168.1.200/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses:
          - 192.168.1.1
          - 8.8.8.8
```

Network settings:

| Setting | Value |
|---|---|
| Interface | `enp5s0` |
| IP address | `192.168.1.200` |
| Subnet | `/24` (`255.255.255.0`) |
| Gateway | `192.168.1.1` |
| Primary DNS | `192.168.1.1` |
| Secondary DNS | `8.8.8.8` |

## 6. Applying Netplan

The configuration was generated/applied successfully using:

```bash
sudo netplan generate
sudo netplan apply
```

The server subsequently reported:

```text
IPv4 address for enp5s0: 192.168.1.200
```

## 7. SSH Verification

SSH access from the Windows PC was successfully tested with:

```bash
ssh anurag@192.168.1.200
```

The connection succeeded and displayed:

```text
Welcome to Ubuntu 24.04.5 LTS
```

This confirms:

- Ethernet networking works.
- The static IP is active.
- SSH is working.
- The server is reachable from the Windows PC.

Normal SSH command going forward:

```bash
ssh anurag@192.168.1.200
```

## 8. SSH Host-Key Message

On the first connection to `192.168.1.200`, SSH displayed the normal message:

```text
The authenticity of host '192.168.1.200' can't be established.
```

SSH also reported that the same host key was known for:

```text
192.168.1.18
192.168.1.28
```

These correspond to the server's previous addresses:

```text
192.168.1.18  -> previous Wi-Fi address
192.168.1.28  -> previous Ethernet DHCP address
192.168.1.200 -> current static Ethernet address
```

The matching host key confirmed that these addresses were associated with the same Ubuntu server.

## 9. Current Network Architecture

```text
                    Internet
                       |
                +------+------+
                | ISP Router  |
                | 192.168.1.1 |
                +------+------+
                       |
                    Ethernet
                       |
                +------+------+
                | Ubuntu      |
                | Server      |
                | enp5s0      |
                | 192.168.1.200
                +-------------+
```

## 10. Current Server Network Baseline

```text
Hostname:       anurag-server
OS:             Ubuntu Server 24.04.5 LTS
Interface:      enp5s0
Connection:     Ethernet
Server IP:      192.168.1.200
Subnet:         192.168.1.0/24
Gateway:        192.168.1.1
DNS:            192.168.1.1, 8.8.8.8
SSH:            Enabled and tested
```

## 11. Future Network Improvements

The current static IP works, but `192.168.1.200` is technically inside the ISP router's stated DHCP range.

### Preferred long-term solution

Contact the ISP and request either:

- DHCP reservation for the server's Ethernet MAC address, or
- A DHCP pool change so the server's static IP is outside the DHCP pool.

Server Ethernet MAC address:

```text
18:c0:4d:21:1f:13
```

An ideal arrangement would be:

```text
Router:       192.168.1.1
DHCP pool:    192.168.1.2 - 192.168.1.100
Server:       192.168.1.200
```

### Alternative: use a personally controlled router

An old personal router can be used to create a separate LAN with a DHCP range under full user control. For example:

```text
ISP Router
192.168.1.1
      |
      | WAN
      v
Personal Router
10.0.0.1
      |
      +---- Ubuntu Server 10.0.0.10
      |
      +---- NAS           10.0.0.11
      |
      +---- Other devices
```

This provides control over DHCP, reservations, static IPs, port forwarding, firewall rules, DNS, and potentially VLANs/network segmentation.

A router behind the ISP router can introduce double NAT. If the ISP router supports bridge mode, that may be preferable for an advanced setup.

## 12. Useful Commands

### Check Ethernet status

```bash
ip addr show enp5s0
```

### Check routing

```bash
ip route
```

### Check router

```bash
ping -c 4 192.168.1.1
```

### Check Internet

```bash
ping -c 4 8.8.8.8
```

### Check DNS

```bash
ping -c 4 google.com
```

### Check Netplan

```bash
sudo cat /etc/netplan/03-lan-static-ip.yaml
```

### Validate Netplan

```bash
sudo netplan generate
```

### Apply Netplan

```bash
sudo netplan apply
```

### SSH into server

```bash
ssh anurag@192.168.1.200
```

## Final Status

**Ubuntu Server Ethernet networking is fully operational.**

The server is currently reachable at:

```text
192.168.1.200
```

SSH access, Internet connectivity, gateway connectivity, and DNS resolution have all been successfully verified.

The remaining networking task is to make the `192.168.1.200` assignment collision-proof through ISP DHCP reservation/pool configuration or by introducing a personally controlled router.
