# Self-Hosted VPN Infrastructure with WireGuard, Docker and Dynamic DNS

End-to-end setup of a self-hosted VPN using WireGuard on Raspberry Pi 5, including CG-NAT bypass, dynamic DNS with DuckDNS, iptables MASQUERADE routing, and secure RDP access to a home network.

## Overview

This project documents the complete process of building a remote access infrastructure from scratch. The goal is to connect via Remote Desktop (RDP) from any external network to a Windows PC on a home network, using a WireGuard VPN running on a Raspberry Pi 5 as the entry point.

The setup covers real-world challenges such as Carrier-Grade NAT (CG-NAT), dynamic IP addresses, subnet conflicts, iptables routing between interfaces, and firewall troubleshooting.

## Architecture

```
Client (outside network)
  -> Internet
    -> Public dynamic IP (ISP)
      -> Router (port forward UDP 51820)
        -> Raspberry Pi 5 (WireGuard in Docker)
          -> Encrypted tunnel established
            -> Client sends traffic to 192.168.50.20
              -> Passes through WireGuard tunnel
                -> Raspberry receives on wg0
                  -> MASQUERADE: changes source from 10.8.0.2 to 192.168.50.244
                    -> Windows PC receives and responds
                      -> Raspberry forwards response through tunnel
                        -> Client receives
                          -> RDP connection established
```

## Requirements

- Raspberry Pi 5 with Raspberry Pi OS
- Router with CG-NAT (tested with Digi Spain)
- Docker and Docker Compose installed on the Raspberry Pi
- DuckDNS account (https://www.duckdns.org)
- WireGuard client on the connecting device (macOS, Windows, iOS, Android)

## 1. CG-NAT Detection and Bypass

### Detecting CG-NAT

Access your router admin panel and check the WAN status. You are behind CG-NAT if:

- The WAN IP address is private (starts with `10.x.x.x`, `100.64-127.x.x`, or `172.16-31.x.x`).
- The default gateway is a private IP address.

With CG-NAT, no inbound connections can reach your network from the internet.

### Solution

Contact your ISP and request a dynamic public IPv4 address. With Digi Spain this is called "Conexion Plus". After activation:

1. Reboot the router.
2. Verify the WAN IP is now public (e.g. `79.x.x.x`).
3. The default gateway should also be a public IP.

If the WAN still shows a private IP after rebooting, contact the ISP again — the change may need manual activation on their end.

## 2. Dynamic DNS with DuckDNS

Since the public IP is dynamic and can change at any time, DuckDNS provides a fixed domain name that always resolves to the current public IP.

1. Create an account at [duckdns.org](https://www.duckdns.org).
2. Create a subdomain (e.g. `mydomain.duckdns.org`).
3. Save your token.

The DuckDNS Docker container handles automatic IP updates.

## 3. Changing the Home Subnet (Recommended)

Most routers default to `192.168.1.0/24`. This causes routing conflicts when connecting from networks that use the same range (offices, hotels, cafes). Traffic intended for the VPN goes to the local network instead.

Change the home subnet to something less common in the router LAN settings:

| Field | Value |
|-------|-------|
| LAN IP | `192.168.50.1` |
| Subnet mask | `255.255.255.0` |
| DHCP range | `192.168.50.128` - `192.168.50.254` |
| Gateway | `192.168.50.1` |
| Primary DNS | `192.168.50.1` |

### Updating the Raspberry Pi 5 Static IP

Raspberry Pi 5 uses NetworkManager. Edit the connection file:

```bash
sudo nano /etc/NetworkManager/system-connections/CONNECTION_NAME.nmconnection
```

Update the `[ipv4]` section:

```ini
[ipv4]
method=manual
address1=192.168.50.244/24,192.168.50.1
dns=192.168.50.1
```

Reboot:

```bash
sudo reboot
```

## 4. Docker Compose — WireGuard and DuckDNS

Create a project directory:

```bash
mkdir ~/docker && cd ~/docker
```

Create a `.env` file with your credentials:

```env
WG_HOST=mydomain.duckdns.org
DD_SUBDOMAINS=mydomain
DD_TOKEN=your_duckdns_token
```

Create a `wg_password_hash.env` file with the wg-easy web interface password hash:

```env
PASSWORD_HASH=your_hash_here
```

Create the `docker-compose.yml`:

```yaml
services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:latest
    container_name: wg-easy
    network_mode: host
    environment:
      LANG: "en"
      WG_HOST: "${WG_HOST}"
      WG_DEFAULT_DNS: "1.1.1.1"
      WG_ALLOWED_IPS: "10.8.0.0/24,192.168.50.0/24"
      WG_POST_UP: "iptables-legacy -t nat -A POSTROUTING -s 10.8.0.0/24 -o wlan0 -j MASQUERADE; iptables-legacy -P FORWARD ACCEPT; iptables -P FORWARD ACCEPT"
      WG_POST_DOWN: "iptables-legacy -t nat -D POSTROUTING -s 10.8.0.0/24 -o wlan0 -j MASQUERADE"
    env_file:
      - wg_password_hash.env
    volumes:
      - ./wireguard:/etc/wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    restart: unless-stopped

  duckdns:
    image: lscr.io/linuxserver/duckdns:latest
    container_name: duckdns
    environment:
      PUID: 1000
      PGID: 1000
      TZ: "Europe/Madrid"
      SUBDOMAINS: "${DD_SUBDOMAINS}"
      TOKEN: "${DD_TOKEN}"
    restart: unless-stopped
```

Start the containers:

```bash
sudo docker compose up -d
```

Notes:
- `network_mode: host` means WireGuard uses the Raspberry Pi network interfaces directly. Ports won't show in `docker ps` — this is expected.
- If the Raspberry Pi is connected via Wi-Fi, the interface is `wlan0`. If connected via Ethernet, change `wlan0` to `eth0` in the `WG_POST_UP` and `WG_POST_DOWN` commands.

## 5. Router Port Forwarding

In the router admin panel, create a port forwarding rule:

| Field | Value |
|-------|-------|
| Protocol | UDP |
| External port | 51820 |
| Internal IP | 192.168.50.244 |
| Internal port | 51820 |

## 6. WireGuard Client Configuration

Install the WireGuard app on your device and create a tunnel:

```ini
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address = 10.8.0.2/24
DNS = 1.1.1.1

[Peer]
PublicKey = SERVER_PUBLIC_KEY
PresharedKey = YOUR_PRESHARED_KEY
AllowedIPs = 10.8.0.0/24, 192.168.50.0/24
Endpoint = mydomain.duckdns.org:51820
PersistentKeepalive = 25
```

Configuration notes:

- `AllowedIPs = 10.8.0.0/24, 192.168.50.0/24` routes only VPN and home network traffic through the tunnel. Use `0.0.0.0/0` to route all traffic through the VPN.
- `PersistentKeepalive = 25` keeps the connection alive when behind NAT.
- `Endpoint` points to the DuckDNS domain which always resolves to the current public IP.

## 7. MASQUERADE — Why It Is Necessary

Without MASQUERADE, VPN clients cannot communicate with devices on the home network. Here is why:

1. The client sends a packet to `192.168.50.20` (Windows PC) with source IP `10.8.0.2`.
2. The packet reaches the Raspberry Pi through the WireGuard tunnel.
3. The Raspberry Pi forwards the packet to the Windows PC.
4. The Windows PC receives a packet from `10.8.0.2`, a network it does not know, and discards it.

With MASQUERADE:

1. The client sends a packet to `192.168.50.20` with source IP `10.8.0.2`.
2. The packet reaches the Raspberry Pi.
3. The Raspberry Pi changes the source IP to its own IP (`192.168.50.244`) before forwarding.
4. The Windows PC receives a packet from `192.168.50.244`, responds to the Raspberry Pi.
5. The Raspberry Pi forwards the response back through the tunnel to the client.

### The Command Explained

```
iptables-legacy -t nat -A POSTROUTING -s 10.8.0.0/24 -o wlan0 -j MASQUERADE
```

| Part | Meaning |
|------|---------|
| `iptables-legacy` | Uses the legacy firewall system (required by WireGuard in Docker) |
| `-t nat` | Operates on the NAT table (IP modification) |
| `-A POSTROUTING` | Applies the rule just before the packet leaves the interface |
| `-s 10.8.0.0/24` | Only applies to packets from the VPN network |
| `-o wlan0` | Only applies to packets going out through Wi-Fi (change to `eth0` for Ethernet) |
| `-j MASQUERADE` | Replaces the source IP with the IP of the outgoing interface |

### Making Rules Persistent

iptables rules are lost on reboot. Two options:

Option 1 — Using wg-easy (recommended): Use `WG_POST_UP` and `WG_POST_DOWN` in `docker-compose.yml` (already included in the configuration above).

Option 2 — Using iptables-persistent:

```bash
sudo apt install iptables-persistent
sudo iptables-legacy-save > /etc/iptables/rules.v4
```

## 8. Remote Desktop (RDP)

With the VPN connected, you can access the Windows PC via RDP.

### Windows PC Setup

1. Enable Remote Desktop: Settings -> System -> Remote Desktop -> On.
2. Allow through firewall:

```cmd
netsh advfirewall firewall add rule name="Allow RDP" protocol=tcp dir=in localport=3389 action=allow
```

### Connecting from macOS

In Microsoft Remote Desktop:

- PC name: `192.168.50.20` (Windows PC local IP)
- Gateway: **None** (important — do not use a Remote Desktop Gateway)
- User account: Your Windows credentials

Using a Remote Desktop Gateway by mistake will result in error `0x300005`. Make sure Gateway is set to **None**.

### Verifying Connectivity

Before connecting via RDP, verify from the client with the VPN active:

```bash
# Ping the Raspberry Pi
ping 192.168.50.244

# Ping the Windows PC
ping 192.168.50.20

# Check RDP port
nc -zv 192.168.50.20 3389
```

## 9. Troubleshooting

### Cannot ping the Raspberry Pi

- Verify WireGuard is running: `sudo docker ps`
- Check port forwarding in the router (UDP 51820 -> 192.168.50.244)
- Verify the handshake: `sudo docker exec wg-easy wg show`
- Check IP forwarding: `cat /proc/sys/net/ipv4/ip_forward` (must be `1`)

### Can ping the Raspberry Pi but not the Windows PC

- MASQUERADE rule is missing. Check with: `sudo iptables-legacy -t nat -L POSTROUTING -v`
- Verify the rule has `pkts > 0` after sending traffic. If `pkts` stays at 0, the rule is not matching.
- Make sure the outgoing interface is correct (`wlan0` for Wi-Fi, `eth0` for Ethernet).
- Check FORWARD policy: `sudo iptables -L FORWARD -v`. If it shows `policy DROP`, run `sudo iptables -P FORWARD ACCEPT`.

### iptables vs iptables-legacy

The Raspberry Pi may have two firewall systems running. WireGuard in Docker uses `iptables-legacy`. Docker itself uses `iptables` (nftables). If rules don't work in one, try the other. Docker sets the FORWARD policy to DROP by default in nftables, which blocks VPN traffic forwarding.

### Subnet Conflict

If the network you are connecting from uses the same range as your home network (e.g. both use `192.168.1.0/24`), traffic will not go through the VPN. The operating system sends packets to the local network instead of the tunnel. Solution: change the home subnet to something uncommon like `192.168.50.0/24`.

You can verify the tunnel itself works by pinging the VPN gateway: `ping 10.8.0.1`. If that responds but `192.168.50.x` addresses do not, it is a subnet conflict.

### Rules Lost After Reboot

Use `WG_POST_UP` in `docker-compose.yml` or install `iptables-persistent`.

## 10. Security

- WireGuard does not respond to packets without valid keys. It is invisible to port scans.
- Never share private keys (PrivateKey, PresharedKey) or your DuckDNS token.
- If keys are compromised, regenerate them:

```bash
wg genkey | tee client_private.key | wg pubkey > client_public.key
wg genpsk > client_preshared.key
```

- Update the client public key in the server configuration and the private key in the client configuration.
- Regenerate the DuckDNS token from the DuckDNS website if exposed.

## Useful Commands

| Command | Purpose |
|---------|---------|
| `sudo docker exec wg-easy wg show` | Check WireGuard status and handshakes |
| `sudo iptables-legacy -t nat -L POSTROUTING -v` | View NAT rules and packet counters |
| `sudo iptables -L FORWARD -v` | Check FORWARD chain policy |
| `curl ifconfig.me` | Check current public IP |
| `nmap -sn 192.168.50.0/24` | Scan local network for devices |
| `nc -zv 192.168.50.20 3389` | Test if RDP port is reachable |
| `udp.port == 51820` | Wireshark filter for WireGuard traffic |
