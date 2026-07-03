# arch-homeserver-vpn-stack

This is my personal Arch Linux home server networking setup.

The main goal of this project was to build a server that I can access remotely, while also keeping some Docker containers routed through a VPN. I wanted to learn how the networking actually works instead of just installing random apps and hoping they work.

The setup uses:

- Arch Linux as the server OS
- Docker and Docker Compose for services
- WireGuard / WG-Easy for VPN access
- Gluetun with ProtonVPN for VPN-isolated container traffic
- DuckDNS for pointing a domain name to my home network

## What I wanted this server to do

I wanted my home server to be able to run normal self-hosted services, but also have a secure way of connecting to it when I am not at home.

The server is on my home LAN:

```text
Home server IP: 192.168.0.123
WireGuard network: 10.8.0.0/24
WG-Easy web UI: 51821/tcp
WireGuard VPN port: 51820/udp
```

The idea is that my laptop or phone can connect to the VPN, then access the server as if I was still at home.

## Why I used WireGuard and WG-Easy

WireGuard is used as the VPN protocol because it is simple, fast and designed around modern cryptography.

WG-Easy runs WireGuard inside Docker and gives me a web UI where I can create a separate client config for each device. This makes it easier to manage devices like:

- laptop
- phone
- tablet
- other machines I want to connect later

Each device gets its own config instead of sharing one config between everything.

## Docker Compose setup

I am using Docker Compose because it keeps the container setup in one file. This makes it easier to see the ports, volumes and settings that each service uses.

Example WG-Easy structure:

```yaml
services:
  wg-easy:
    container_name: wg-easy
    image: ghcr.io/wg-easy/wg-easy

    environment:
      - WG_HOST=example.duckdns.org
      - PASSWORD_HASH=CHANGE_ME

    volumes:
      - ./config:/etc/wireguard
      - /lib/modules:/lib/modules

    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"

    restart: unless-stopped

    cap_add:
      - NET_ADMIN
      - SYS_MODULE

    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
```

The important part here is that WireGuard uses UDP port `51820`, while the WG-Easy web interface uses TCP port `51821`.

## DuckDNS

I used DuckDNS so my VPN config can point to a domain name instead of only using my public IP address.

Instead of putting a changing public IP into the WireGuard client config, I can use something like:

```text
example.duckdns.org
```

Then the WireGuard client can use:

```text
Endpoint = example.duckdns.org:51820
```

This is useful because home public IP addresses can change.

## Split tunnel client config

For the laptop, I wanted a split tunnel setup.

This means only the home server / home LAN traffic goes through the VPN. Normal websites still use the laptop's normal internet connection.

Example client config:

```ini
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.8.0.2/24

[Peer]
PublicKey = SERVER_PUBLIC_KEY
PresharedKey = PRESHARED_KEY
AllowedIPs = 10.8.0.0/24, 192.168.0.0/24
PersistentKeepalive = 25
Endpoint = example.duckdns.org:51820
```

The important part is:

```ini
AllowedIPs = 10.8.0.0/24, 192.168.0.0/24
```

This means:

```text
10.8.0.0/24    = WireGuard VPN network
192.168.0.0/24 = home LAN
```

It does not force every website through the home server.

## Gluetun and ProtonVPN

I also use Gluetun with ProtonVPN for containers that need their traffic routed through a VPN.

This is separate from the WireGuard remote access setup.

The way I think about it is:

```text
WireGuard / WG-Easy:
outside device -> home server

Gluetun / ProtonVPN:
selected Docker container -> ProtonVPN -> internet
```

This lets the server still host normal services, while only specific containers use the ProtonVPN connection.

Example idea:

```yaml
services:
  gluetun:
    image: qmcgaw/gluetun
    container_name: gluetun

    cap_add:
      - NET_ADMIN

    environment:
      - VPN_SERVICE_PROVIDER=protonvpn
      - VPN_TYPE=wireguard
      - WIREGUARD_PRIVATE_KEY=CHANGE_ME

    restart: unless-stopped

  example-container:
    image: example/image
    network_mode: "service:gluetun"
    depends_on:
      - gluetun
```

The important line is:

```yaml
network_mode: "service:gluetun"
```

That means the container shares Gluetun's network connection, so its traffic goes through the VPN container.

## Things I learned

This project helped me understand:

- public IP addresses vs private LAN IP addresses
- how port forwarding works
- why WireGuard uses UDP
- how Docker exposes ports
- how Docker Compose makes services easier to manage
- how VPN routing works
- what split tunnelling means
- how to separate remote access traffic from VPN-isolated container traffic
- how to debug networking using commands like `wg show`, `ip route`, `ss`, and `tcpdump`

## Useful commands

Check running containers:

```bash
docker ps
```

Check WG-Easy ports:

```bash
docker ps --format "table {{.Names}}\t{{.Ports}}"
```

Check WireGuard inside the container:

```bash
docker exec -it wg-easy wg show
```

Check if the server is listening on UDP 51820:

```bash
sudo ss -lunp | grep 51820
```

Check the server IP:

```bash
ip -4 -br addr
```

Check if packets are reaching the server:

```bash
sudo tcpdump -ni wlp2s0 udp port 51820
```

Start the stack:

```bash
docker compose up -d
```

Stop the stack:

```bash
docker compose down
```

## Security notes

I do not commit real private keys, preshared keys, VPN tokens or passwords to this repository.

Anything sensitive should be replaced with placeholders like:

```text
CHANGE_ME
PRIVATE_KEY_HERE
TOKEN_HERE
PASSWORD_HASH_HERE
```

## Future improvements

Things I want to improve later:

- move more secrets into `.env` files
- add better firewall rules
- document the full Docker stack
- add monitoring for uptime
- move the server from Wi-Fi to Ethernet for better reliability
- add backups for Docker config files
- clean up the WireGuard client configs
- make the setup easier to rebuild from scratch

## Summary

This project is my attempt at building a proper Arch Linux home server network stack.

It combines Docker, WireGuard, WG-Easy, DuckDNS and Gluetun so I can learn more about real Linux networking and self-hosting. The main focus is remote access, container management and routing selected containers through a VPN without forcing the whole server behind it.
