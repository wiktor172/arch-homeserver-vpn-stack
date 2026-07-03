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

The server is on my home LAN
