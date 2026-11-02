---
layout: post
title: "Setting Up a VPN on Linux: WireGuard and OpenVPN Compared"
date: Nov-08-2012
categories: [Linux, Networking, Security]
tags: [linux, vpn, wireguard, openvpn, security]
image: /assets/img/vpn-security-padlock-chain.jpg
---

A VPN on Linux usually means one of two things today: WireGuard, a small, modern kernel-integrated protocol, or OpenVPN, the older and more flexible but heavier option. This post walks through a minimal WireGuard peer-to-peer setup, contrasts it with an OpenVPN certificate-based setup, and covers the firewall/NAT pieces either one needs to actually route traffic.

## Choosing Between WireGuard and OpenVPN

```bash
# WireGuard: simpler config, modern crypto, in-kernel since Linux 5.6,
# UDP-only, authenticates with keypairs rather than a PKI
#
# OpenVPN: older, more features (TCP or UDP, routing options, plugins),
# heavier setup, uses a certificate authority for trust
```

*Tip:* Reach for WireGuard by default for a site-to-site tunnel or a personal VPN; reach for OpenVPN when you specifically need TCP mode to get through a restrictive firewall, or need its more elaborate per-client access control.

## WireGuard: Generating Keypairs

Every WireGuard peer, server or client, has its own keypair; there's no shared certificate authority to manage.

```bash
sudo apt install wireguard

# Generate a private key and derive the matching public key
wg genkey | tee server_private.key | wg pubkey > server_public.key
wg genkey | tee client_private.key | wg pubkey > client_public.key

cat server_private.key
# <base64 private key>
cat server_public.key
# <base64 public key, this is what you share>
```

*Tip:* A WireGuard private key never needs to be shared with the other peer — only the public key does. Treat the private key file with the same care as an SSH private key.

## A Minimal Server Configuration

```ini
# /etc/wireguard/wg0.conf, on the server
[Interface]
Address = 10.8.0.1/24
ListenPort = 51820
PrivateKey = <server_private.key contents>

[Peer]
PublicKey = <client_public.key contents>
AllowedIPs = 10.8.0.2/32
```

```ini
# /etc/wireguard/wg0.conf, on the client
[Interface]
Address = 10.8.0.2/24
PrivateKey = <client_private.key contents>

[Peer]
PublicKey = <server_public.key contents>
Endpoint = vpn.example.com:51820
AllowedIPs = 10.8.0.0/24
PersistentKeepalive = 25
```

*Tip:* `AllowedIPs` does double duty: on the server it also acts as a routing table entry (which source addresses are allowed to send from each peer), and on the client it controls which destination traffic gets sent through the tunnel at all.

## Bringing the Tunnel Up

```bash
# Bring the interface up on both ends
sudo wg-quick up wg0

# Enable it at boot
sudo systemctl enable wg-quick@wg0

# Tear it down
sudo wg-quick down wg0
```

*Tip:* `wg-quick` is a convenience wrapper that also handles routing table entries and, if configured, firewall rules via `PostUp`/`PostDown` lines in the config — the bare `wg` command only manages the interface and peers themselves.

## Checking Tunnel State with wg show

```bash
sudo wg show
# interface: wg0
#   public key: <server_public_key>
#   private key: (hidden)
#   listening port: 51820
#
# peer: <client_public_key>
#   endpoint: 203.0.113.5:54821
#   allowed ips: 10.8.0.2/32
#   latest handshake: 42 seconds ago
#   transfer: 1.21 MiB received, 3.44 MiB sent
```

*Tip:* "latest handshake" is the single most useful line here — WireGuard re-handshakes roughly every two minutes while traffic flows, so a handshake that's old or missing means the tunnel isn't actually passing traffic, regardless of what the interface status shows.

## OpenVPN: A Certificate-Based Setup, in Contrast

OpenVPN's trust model uses a certificate authority: the server and every client get a certificate signed by the same CA, and the server verifies client certs against it.

```bash
sudo apt install openvpn easy-rsa

# Build a CA and server/client certs (easy-rsa, abbreviated)
make-cadir ~/openvpn-ca && cd ~/openvpn-ca
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa build-server-full server nopass
./easyrsa build-client-full client1 nopass
```

```
# /etc/openvpn/server.conf (key lines only)
port 1194
proto udp
dev tun
ca ca.crt
cert server.crt
key server.key
dh dh.pem
server 10.9.0.0 255.255.255.0
```

```bash
sudo systemctl start openvpn@server
```

*Tip:* The certificate model is what makes OpenVPN's per-client revocation possible — `easyrsa revoke client1` plus a distributed CRL cuts off one compromised client without touching anyone else's config, something WireGuard's static `AllowedIPs` peer list doesn't offer as cleanly on its own.

## Firewall and NAT: Making the Tunnel Actually Route Traffic

A VPN interface coming up is not the same as traffic being allowed to flow through it; both ends typically need explicit firewall and forwarding rules.

```bash
# Enable IP forwarding on the VPN server
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# NAT traffic from the VPN subnet out through the server's main interface,
# so clients can reach the internet (or the internal LAN) through the tunnel
sudo nft add table ip nat
sudo nft add chain ip nat postrouting { type nat hook postrouting priority 100 \; }
sudo nft add rule ip nat postrouting ip saddr 10.8.0.0/24 oifname "eth0" masquerade

# Allow the VPN port through the firewall
sudo nft add rule inet filter input udp dport 51820 accept
```

*Tip:* `ip_forward=1` alone doesn't help if the firewall's forward chain defaults to drop — check both the sysctl setting and the actual `FORWARD` policy/rules, since either one missing results in the same symptom: a tunnel that comes up but can't reach anything past the server itself.
