# us1 Home Lab Server

## Overview

`us1` is an Ubuntu Server VM hosted on Proxmox and provides Docker services, file sharing, SSH administration, and secure remote access to the home network through WireGuard.

This document records the working network, firewall, WireGuard, routing, and recovery configuration.

## Network Layout

### us1

- Hostname: `us1`
- OS: Ubuntu 26.04.1 LTS
- LAN interface: `ens18`
- LAN IP: `192.168.4.42/22`
- LAN subnet: `192.168.4.0/22`
- Default gateway: `192.168.4.1`

### Proxmox

- Hostname: `proxmox2`
- LAN IP: `192.168.4.220`
- Web interface: `https://192.168.4.220:8006`

### Docker Networks

Observed Docker bridge networks:

- `172.17.0.0/16`
- `172.18.0.0/16`
- `172.19.0.0/16`

### WireGuard

- Interface: `wg0`
- VPN subnet: `10.44.0.0/24`
- us1 VPN address: `10.44.0.1/24`
- iPhone VPN address: `10.44.0.2/32`
- Protocol: UDP
- Server port: `51820`
- Mode: Split tunnel

The iPhone routes only these networks through WireGuard:

- `10.44.0.0/24`
- `192.168.4.0/22`

Normal iPhone Internet traffic does not use the home VPN.

---

# Network Validation

The following connectivity was tested successfully from `us1`.

## Gateway

```bash
ping -c 4 192.168.4.1
```

Result:

- 0% packet loss

## IPv4 Internet

```bash
ping -c 4 1.1.1.1
```

Result:

- 0% packet loss

## DNS and IPv6 Internet

```bash
ping -c 4 google.com
```

`google.com` resolved to IPv6 and responded successfully.

---

# Listening Services

Check listening ports with:

```bash
sudo ss -tulpn
```

Expected externally reachable services on the LAN include:

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH |
| 137 | UDP | NetBIOS/Samba |
| 138 | UDP | NetBIOS/Samba |
| 139 | TCP | Samba |
| 445 | TCP | SMB |
| 8080 | TCP | Docker web service |
| 8083 | TCP | Docker web service |
| 51820 | UDP | WireGuard |

TCP LAN connectivity was verified from `proxmox2` with:

```bash
nc -zv 192.168.4.42 22 139 445 8080 8083
```

---

# UFW Firewall

UFW is enabled and starts automatically.

Check status with:

```bash
sudo ufw status verbose
```

Current policy:

```text
Default: deny (incoming), allow (outgoing), deny (routed)
```

## LAN Rules

SSH is allowed only from the home LAN:

```bash
sudo ufw allow from 192.168.4.0/22 to any port 22 proto tcp
```

Samba is allowed only from the home LAN:

```bash
sudo ufw allow from 192.168.4.0/22 to any port 139 proto tcp
sudo ufw allow from 192.168.4.0/22 to any port 445 proto tcp
sudo ufw allow from 192.168.4.0/22 to any port 137 proto udp
sudo ufw allow from 192.168.4.0/22 to any port 138 proto udp
```

Docker web services are allowed only from the home LAN:

```bash
sudo ufw allow from 192.168.4.0/22 to any port 8080 proto tcp
sudo ufw allow from 192.168.4.0/22 to any port 8083 proto tcp
```

## WireGuard Rule

WireGuard must accept connections from outside the home network:

```bash
sudo ufw allow 51820/udp comment 'WireGuard'
```

This creates IPv4 and IPv6 rules.

## WireGuard-to-LAN Routing

WireGuard clients are permitted to route to the home LAN:

```bash
sudo ufw route allow in on wg0 out on ens18 from 10.44.0.0/24 to 192.168.4.0/22
```

The global routed policy remains `deny`.

---

# WireGuard Server

WireGuard packages installed:

```bash
sudo apt install wireguard
```

Verify:

```bash
wg --version
```

Configuration directory:

```text
/etc/wireguard/
```

Important files:

```text
server_private.key
server_public.key
phone_private.key
phone_public.key
wg0.conf
phone.conf
```

## IMPORTANT: Secrets

The following files contain sensitive key material and MUST NOT be committed to GitHub:

```text
server_private.key
phone_private.key
phone.conf
wg0.conf
```

Treat a WireGuard QR code containing a client configuration as a secret as well.

---

# WireGuard Server Configuration

Actual configuration:

```text
/etc/wireguard/wg0.conf
```

Conceptually the configuration is:

```ini
[Interface]
Address = 10.44.0.1/24
ListenPort = 51820
PrivateKey = SERVER_PRIVATE_KEY_REDACTED

[Peer]
# Phone
PublicKey = PHONE_PUBLIC_KEY
AllowedIPs = 10.44.0.2/32
```

Never put the real server private key in documentation or GitHub.

Permissions:

```bash
chmod 600 /etc/wireguard/wg0.conf
```

---

# iPhone WireGuard Configuration

The iPhone configuration is stored securely as:

```text
/etc/wireguard/phone.conf
```

Conceptual configuration:

```ini
[Interface]
PrivateKey = PHONE_PRIVATE_KEY_REDACTED
Address = 10.44.0.2/32

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = PUBLIC_IP_REDACTED:51820
AllowedIPs = 10.44.0.0/24, 192.168.4.0/22
PersistentKeepalive = 25
```

The configuration is intentionally split tunnel.

Only traffic for:

```text
10.44.0.0/24
192.168.4.0/22
```

travels through WireGuard.

---

# Router Configuration

The Internet router forwards:

```text
UDP 51820
    |
    v
192.168.4.42:51820
```

Only UDP `51820` is forwarded for WireGuard.

Do not create Internet port forwards for:

- SSH `22`
- SMB `139`
- SMB `445`
- Docker `8080`
- Docker `8083`
- Proxmox `8006`

These services should be accessed remotely through WireGuard instead.

---

# IP Forwarding

IPv4 forwarding is enabled.

Verify with:

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

---

# WireGuard-to-LAN NAT

WireGuard clients require a return path from LAN devices.

The following NAT configuration was added to:

```text
/etc/ufw/before.rules
```

It appears before the existing `*filter` table:

```text
# WireGuard VPN to home LAN NAT
*nat
:POSTROUTING ACCEPT [0:0]
-A POSTROUTING -s 10.44.0.0/24 -d 192.168.4.0/22 -o ens18 -j MASQUERADE
COMMIT
```

A backup was created before editing:

```text
/etc/ufw/before.rules.pre-wireguard
```

Validate the UFW rules before reloading them:

```bash
sudo iptables-restore --test < /etc/ufw/before.rules
```

No output indicates successful syntax validation.

Reload UFW:

```bash
sudo ufw reload
```

Verify the NAT rule:

```bash
sudo iptables -t nat -S POSTROUTING | grep 10.44.0.0/24
```

Expected:

```text
-A POSTROUTING -s 10.44.0.0/24 -d 192.168.4.0/22 -o ens18 -j MASQUERADE
```

---

# Automatic WireGuard Startup

WireGuard starts automatically using systemd:

```bash
sudo systemctl enable wg-quick@wg0
```

Verify:

```bash
systemctl is-enabled wg-quick@wg0
```

Expected:

```text
enabled
```

Check service state:

```bash
systemctl status wg-quick@wg0 --no-pager
```

`active (exited)` is the expected state after `wg-quick` successfully creates and configures the interface.

---

# WireGuard Verification

Check WireGuard status:

```bash
sudo wg show
```

Expected interface information:

```text
interface: wg0
listening port: 51820

peer: PHONE_PUBLIC_KEY
allowed ips: 10.44.0.2/32
```

When the phone is connected, additional fields should appear including:

```text
endpoint
latest handshake
transfer
```

Check the interface address:

```bash
ip -br addr show wg0
```

Expected:

```text
wg0    UNKNOWN    10.44.0.1/24
```

---

# Remote Access Test

On the iPhone:

1. Disable Wi-Fi.
2. Use cellular data.
3. Enable the `Home VPN` WireGuard tunnel.

The following was successfully tested through WireGuard:

```text
http://10.44.0.1:8080
```

The Proxmox interface was also successfully reached through the VPN:

```text
https://192.168.4.220:8006
```

This verifies the complete path:

```text
iPhone
  -> Cellular Internet
  -> Home Router
  -> UDP 51820
  -> us1
  -> WireGuard wg0
  -> UFW forwarding
  -> NAT
  -> Home LAN
  -> Proxmox
```

---

# Reboot Persistence Test

After rebooting `us1`, the following were confirmed:

- UFW started automatically.
- `wg0` started automatically.
- `wg0` received `10.44.0.1/24`.
- The phone peer was loaded.
- The WireGuard-to-LAN forwarding rule remained.
- The NAT masquerade rule remained.
- WireGuard listened on UDP `51820`.

Useful post-reboot verification:

```bash
systemctl status wg-quick@wg0 --no-pager
sudo ufw status
sudo wg show
ip -br addr show wg0
sudo iptables -t nat -S POSTROUTING | grep 10.44.0.0/24
```

---

# Public Exposure Testing

Before configuring WireGuard, public IPv4 and IPv6 tests were performed against:

```text
22
139
445
8080
8083
```

All returned no response/filtered.

After UFW was configured, the external tests were repeated and continued to return no response.

UDP `51820` is intentionally the public entry point for WireGuard.

---

# Adding Another WireGuard Client

Each device should have its own key pair and its own VPN IP.

Do not reuse the iPhone private key on additional devices.

Example addressing plan:

```text
10.44.0.1    us1 WireGuard server
10.44.0.2    iPhone
10.44.0.3    next client
10.44.0.4    next client
...
```

On the server, each client receives its own peer entry:

```ini
[Peer]
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 10.44.0.X/32
```

Each client's private key stays on that client.

---

# Lost or Compromised Device

If a WireGuard client is lost or compromised, remove its peer from:

```text
/etc/wireguard/wg0.conf
```

Then reload the active configuration:

```bash
sudo wg syncconf wg0 <(sudo wg-quick strip wg0)
```

Verify:

```bash
sudo wg show
```

The compromised peer should no longer appear.

Do not simply assign the same private key to a replacement device. Generate a new client key pair.

---

# Troubleshooting

## Check WireGuard

```bash
sudo wg show
```

Look for:

- Correct peer
- Recent handshake
- Transfer counters

## Check WireGuard interface

```bash
ip -br addr show wg0
```

## Check listening port

```bash
sudo ss -ulpn | grep 51820
```

## Check UFW

```bash
sudo ufw status verbose
```

## Check IP forwarding

```bash
sysctl net.ipv4.ip_forward
```

## Check LAN route

```bash
ip route get 192.168.4.220
```

Expected interface:

```text
ens18
```

## Check NAT

```bash
sudo iptables -t nat -S POSTROUTING | grep 10.44.0.0/24
```

## Restart WireGuard

```bash
sudo systemctl restart wg-quick@wg0
```

Then:

```bash
sudo wg show
```

---

# Recovery

A Proxmox snapshot was taken before the UFW/WireGuard configuration changes.

There is also a backup of the original UFW rules file:

```text
/etc/ufw/before.rules.pre-wireguard
```

If the NAT configuration causes problems, restore the previous rules file:

```bash
sudo cp /etc/ufw/before.rules.pre-wireguard /etc/ufw/before.rules
```

Validate:

```bash
sudo iptables-restore --test < /etc/ufw/before.rules
```

Then reload:

```bash
sudo ufw reload
```

If networking becomes inaccessible remotely, use the Proxmox console rather than making additional firewall changes blindly.

---

# Recommended Git Repository Protection

A documentation repository should include a `.gitignore` that prevents WireGuard secrets and environment files from being committed.

Recommended entries:

```gitignore
# WireGuard secrets
*_private.key
phone.conf
wg0.conf
*.key

# Environment/secrets
.env
.env.*
*.env

# Certificates/private material
*.pem
*.p12
*.pfx

# Backup files that might contain secrets
*.bak
*.backup
```

Before every push involving security configuration, check:

```bash
git status
git diff --cached
```

If a private key is ever accidentally committed, removing the file in a later commit is not sufficient. Treat the exposed key as compromised, generate a replacement, update the affected WireGuard peer, and then clean the repository history as necessary.

---

# Backup Strategy

Recommended protection consists of three separate layers:

1. **Proxmox backup**
   - Protects the complete `us1` VM.

2. **Private Git repository**
   - Stores documentation, redacted configuration examples, procedures, and change history.

3. **Secure secret backup**
   - Separately protects WireGuard private keys and other credentials.
   - Secrets are never stored in the Git repository.

A permanent Proxmox VM backup should be created after the configuration has been proven stable.

Suggested backup description:

```text
us1-post-wireguard-ufw
```

---

# Current Working State

At the completion of this configuration:

- `us1` networking is healthy.
- IPv4 Internet works.
- IPv6 Internet works.
- DNS works.
- UFW is active.
- Incoming traffic is denied by default.
- SSH is LAN-only.
- SMB is LAN-only.
- Docker web services are LAN-only.
- WireGuard UDP `51820` is the remote VPN entry point.
- WireGuard starts automatically.
- The iPhone connects successfully over cellular.
- The VPN uses split tunneling.
- Remote access to Proxmox through WireGuard works.
- WireGuard-to-LAN routing persists across reboot.
- WireGuard-to-LAN NAT persists across reboot.
- A pre-change Proxmox snapshot exists.

## Security Principle

**Do not expose individual home-lab management services to the Internet when they can instead be reached through WireGuard.**

Remote administration should enter the home network through the VPN and then access internal services over their normal LAN addresses.
