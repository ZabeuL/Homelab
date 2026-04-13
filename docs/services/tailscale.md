# Tailscale

Installed as a systemd service on the Ubuntu VM (not a Docker container). Provides remote access to the homelab over WireGuard without exposing any ports on the Bell Gigahub.

## Installation

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

## Startup Command

```bash
sudo tailscale up --advertise-routes=192.168.2.0/24 --accept-dns=false
```

- `--advertise-routes=192.168.2.0/24` — makes all LAN devices reachable remotely by their local IPs without installing Tailscale on each one
- `--accept-dns=false` — prevents Tailscale from overriding the VM's DNS with MagicDNS, which would break AdGuard resolution on the VM itself

## IP Forwarding

Required for subnet routing — the VM must forward packets between the Tailscale interface and the LAN.

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

## Admin Console Configuration

- **Subnet routes:** Approve `192.168.2.0/24` under Machines → Edit route settings
- **Global nameservers:** VM's Tailscale IP (`100.x.x.x`) as primary, `9.9.9.9` as fallback, with Override DNS enabled — routes all DNS through AdGuard when connected
- **MagicDNS:** Left enabled — handles Tailscale device name resolution, does not conflict with AdGuard

## Result

When connected via Tailscale on any device, all `192.168.2.x` addresses are reachable and `*.zabeu.dev` domains resolve correctly via AdGuard — identical behavior to being on the local network.
