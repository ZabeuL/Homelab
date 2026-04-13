# Home Lab

A phased home lab build on recycled desktop hardware, with two goals: practical home use (ad blocking, photo management, cameras, remote access) and hands-on experience with infrastructure and networking.

## Architecture

```
[Bell Gigahub 2.0]
     │
     │  2.5GbE Cat6
     │
  [Server]  192.168.2.2
   Proxmox VE
   └── Ubuntu Server 24.04 VM  192.168.2.3
            └── Docker Compose
               ├── AdGuard Home          (DNS + DHCP + ad blocking)
               ├── Nginx Proxy Manager   (reverse proxy + SSL)
               └── Tailscale             (remote VPN access)
```

| Device | IP | Type |
|---|---|---|
| Bell Gigahub | `192.168.2.1` | Router (fixed) |
| Proxmox host | `192.168.2.2` | Static |
| Ubuntu VM | `192.168.2.3` | Static |
| DHCP pool | `192.168.2.10–254` | Dynamic (issued by AdGuard) |

## Phases

| Phase | Status | Goal |
|---|---|---|
| **Phase 1** | ✅ Complete | Proxmox + Docker Compose services (AdGuard, NPM, Tailscale) |
| **Phase 2** | 🔜 Planned | NAS with ZFS mirror, Immich photo management, SMB shares |
| **Phase 3** | 🔜 Planned | Reolink PoE cameras, Frigate NVR with Coral USB TPU |

## Docs

- [Hardware](docs/hardware.md)
- [Architecture & Design Decisions](docs/architecture.md)
- [Proxmox](docs/proxmox.md)
- [Ubuntu Server VM](docs/ubuntu-vm.md)
- [AdGuard Home](docs/services/adguard.md)
- [Nginx Proxy Manager](docs/services/nginx-proxy-manager.md)
- [Tailscale](docs/services/tailscale.md)
