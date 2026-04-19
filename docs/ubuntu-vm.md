# Ubuntu Server VM

**OS:** Ubuntu Server 24.04 LTS  
**IP:** `192.168.2.3` (static via Netplan)  
**Resources:** 2 vCPU, 4GB RAM, 60GB disk on local storage  
**NIC:** `ens18`

## Why a VM Instead of LXC

Docker networking features — particularly `network_mode: host` for AdGuard DHCP — don't work cleanly inside a Proxmox LXC without enabling privileged mode. A full VM gives Docker a complete, isolated Linux environment with no kernel-sharing edge cases. See [Architecture](architecture.md) for more detail.

## Key Install Decisions

- **No LVM on the VM disk** — Proxmox manages the virtual disk directly; LVM inside the VM adds complexity with no benefit at this scale
- **Static IP assigned post-install** — DHCP address assigned during the Ubuntu installer; static assignment configured via Netplan after Docker was running

## Static IP (Netplan)

`/etc/netplan/50-cloud-init.yaml`:

```yaml
network:
  version: 2
  ethernets:
    ens18:
      dhcp4: false
      addresses:
        - 192.168.2.3/24
      routes:
        - to: default
          via: 192.168.2.2
      nameservers:
        addresses:
          - 192.168.2.3
          - 9.9.9.9
```

Gateway is `192.168.2.2` (Proxmox), not `192.168.2.1` (router) — the Bell Gigahub ignores ARP from MACs it did not assign via DHCP. Proxmox NAT-masquerades the VM's outbound traffic. See [Proxmox](proxmox.md) and [Architecture](architecture.md).

The VM cannot directly ping `192.168.2.1` — this is expected.

## Docker Install

Installed via Docker's official apt repository: https://docs.docker.com/engine/install/ubuntu/

**Post-install:**

```bash
# Avoid requiring sudo for every Docker command
sudo usermod -aG docker $USER
```

**Log rotation** (`/etc/docker/daemon.json`):

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Max 30MB of logs per container (3 × 10MB rotating files).

## Docker Compose

All services are defined in `/opt/homelab/docker-compose.yml`. Bind mounts use subdirectories under `/opt/homelab/` so config files are directly accessible on disk.

## Claude Code

Installed on the Ubuntu VM as the CLI tool for running AI-assisted tasks directly on the server.

**Install:**

```bash
npm install -g @anthropic-ai/claude-code
```

**Purpose:** Syncs Notion documentation to the repo's markdown files on demand. Claude Code reads a Notion integration token stored locally, fetches the relevant pages via the Notion REST API, and rewrites the corresponding `.md` files under `/opt/homelab/docs/`.

**Token storage:** `/opt/homelab/personal/notion-token` — this directory is gitignored and never pushed to GitHub.

**Usage:** Triggered manually after completing a phase or resolving a notable issue. Not automated.
