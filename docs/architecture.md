# Architecture & Design Decisions

## Why Proxmox Over TrueNAS

TrueNAS Scale bundles a hypervisor but is primarily a storage OS. Proxmox is a general-purpose hypervisor that supports both VMs and LXC containers, is widely used in production and homelabs, and maps more directly to skills applicable in IT and infrastructure roles. Storage (ZFS, NAS) is added in Phase 2 as managed disk passthrough, not as the foundation.

## Why a VM Instead of LXC for Docker

Docker networking features — particularly `network_mode: host` and DHCP server use cases — don't work cleanly inside a Proxmox LXC without enabling privileged mode. A full VM gives Docker a complete, isolated Linux environment with no kernel-sharing edge cases. This is also the pattern used in production infrastructure.

## Why Docker Compose Over Kubernetes

Docker Compose before Kubernetes is an intentional learning progression. Compose is the right tool at this scale and teaches container orchestration fundamentals without the overhead of managing a cluster.

## Bell Gigahub Constraints

The Bell Gigahub 2.0 is ISP-provided and significantly locked down:

- **No custom DHCP DNS option** — the DHCP settings panel has no field to push a custom DNS server to clients. The router always advertises itself (`192.168.2.1`) as DNS.
- **No local IP as upstream DNS** — the router rejects local addresses (e.g. `192.168.2.3`) as the upstream DNS setting.
- **DNS proxy cannot be disabled** — all DNS queries sent to `192.168.2.1` are intercepted and resolved by the router. There is no passthrough option.
- **ARP table only includes DHCP-assigned MACs** — the Gigahub only builds ARP entries for devices it assigned via DHCP. A statically configured device is invisible to the router at layer 2; the router ignores ARP requests from that MAC entirely.

These constraints drove two key architectural decisions:

1. **AdGuard as DHCP server** — the only way to push a custom DNS server to LAN clients is to replace the Gigahub as the DHCP authority. AdGuard issues all leases and advertises itself as DNS.
2. **Proxmox as NAT gateway for the VM** — since the VM's MAC is not in the Gigahub's ARP table, it cannot reach the router directly. Proxmox masquerades the VM's outbound traffic as its own IP (`192.168.2.2`), which the router does recognize. The VM's default gateway is set to `192.168.2.2`, not `192.168.2.1`. Any additional VMs added to this lab should follow the same pattern.

See [Proxmox](proxmox.md) for the NAT configuration.

## DNS & Routing Flow

```
LAN client
  │
  ├─ DHCP lease ──────────────────────► AdGuard (192.168.2.3)
  │                                      └─ issues IP + DNS server
  │
  ├─ DNS query (port 53) ─────────────► AdGuard (192.168.2.3)
  │                                      └─ DoH upstream → Quad9 / Cloudflare
  │
  └─ HTTPS request (*.zabeu.dev) ─────► NPM (192.168.2.3:443)
                                         └─ routes to backend by domain name
```

AdGuard DNS rewrites resolve all `*.zabeu.dev` subdomains to `192.168.2.3` (NPM). NPM terminates SSL and forwards to the correct backend based on the domain name. SSL is a wildcard Let's Encrypt cert issued via DNS challenge — no ports exposed to the internet.

## Storage Layout (Proxmox)

`local-lvm` was merged into `local` storage given the modest SSD size. At this scale, separating VM disk storage from ISO/backup storage adds complexity with no benefit.

```bash
lvremove /dev/pve/data
lvresize -l +100%FREE /dev/pve/root
resize2fs /dev/mapper/pve-root
```
