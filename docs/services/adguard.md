# AdGuard Home

Runs as a Docker container on the Ubuntu VM with host networking. Serves as both the network-wide DNS resolver and DHCP server — all devices receive their IP leases and DNS server assignment directly from AdGuard.

## Docker Compose

```yaml
services:
  adguard:
    image: adguard/adguardhome:latest
    container_name: adguard-home
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./adguard/work:/opt/adguardhome/work
      - ./adguard/conf:/opt/adguardhome/conf
```

`network_mode: host` is required for DHCP — see [Why host networking](#why-host-networking-is-required-for-dhcp) below. With host mode, the `ports:` block is omitted entirely — port mappings are ignored in host mode. AdGuard binds directly to the VM's interfaces.

## DHCP Configuration

Set under AdGuard UI → Settings → DHCP. The Bell Gigahub's DHCP is disabled — AdGuard is the sole DHCP server on the network.

| Field | Value |
|---|---|
| Interface | `ens18` |
| IP Range | `192.168.2.10` – `192.168.2.254` |
| Subnet Mask | `255.255.255.0` |
| Gateway | `192.168.2.1` |
| DNS Server (primary) | `192.168.2.3` (AdGuard itself) |
| DNS Server (fallback) | `9.9.9.9` (Quad9) |
| Lease Duration | `24h` |

The fallback DNS (`9.9.9.9`) is pushed to clients via DHCP option 6 in `AdGuardHome.yaml`. If AdGuard goes down, clients fall back to Quad9 automatically and internet access continues — ad blocking stops but connectivity doesn't.

**Note:** `nslookup` on Windows does not use the DHCP-provided fallback. Use `ping` or a browser to verify fallback behavior.

## DNS Configuration

- **Upstream DNS:** DNS-over-HTTPS (DoH) for encrypted upstream resolution
  - Primary: `https://dns.quad9.net/dns-query`
  - Secondary: `https://cloudflare-dns.com/dns-query`
- AdGuard queries both upstreams in parallel and uses the fastest response
- Clients send plain DNS to `192.168.2.3`; AdGuard encrypts all upstream queries via DoH

**Blocklists active:** Hagezi Pro, TIF

## DNS Rewrites

All `*.zabeu.dev` subdomains resolve to `192.168.2.3` (NPM). Add one rewrite per service under Filters → DNS Rewrites.

| Domain | Answer |
|---|---|
| `proxmox.zabeu.dev` | `192.168.2.3` |
| `adguard.zabeu.dev` | `192.168.2.3` |
| `npm.zabeu.dev` | `192.168.2.3` |

NPM reads the domain name from the request and routes to the correct backend. See [Nginx Proxy Manager](nginx-proxy-manager.md).

## Why Host Networking Is Required for DHCP

Docker's default bridge networking uses iptables NAT to forward traffic into containers. This works for TCP/UDP unicast (DNS, HTTP) but fails for DHCP for two reasons:

- **DHCP is a broadcast protocol.** Clients with no IP send discover packets to `255.255.255.255`. Docker's iptables NAT rules only intercept directed (unicast) traffic — broadcasts are never forwarded into the container.
- **A DHCP relay also fails in this architecture.** `isc-dhcp-relay` on Proxmox can convert LAN broadcasts into unicast packets aimed at `192.168.2.3`, but Docker's bridge iptables rules intercept that unicast at the host before it reaches the container. The relay ran correctly (`dhcrelay -q -i vmbr0 192.168.2.3`) but the DHCP handshake still broke.

With `network_mode: host`, the container shares the VM's network stack directly. AdGuard binds to `ens18` as if it were a native process — no port mapping, no NAT. Broadcast DHCP discovers arrive on `ens18` and AdGuard receives them. The tradeoff is that host mode removes Docker's network isolation for this container, which is acceptable since AdGuard intentionally needs to be on the LAN.

## Why AdGuard Is the DHCP Server

The Bell Gigahub 2.0 has no option to push a custom DNS server to DHCP clients — it always advertises itself (`192.168.2.1`) as DNS and resolves queries itself. There is no way to disable this or configure a local upstream. As long as the Gigahub is the DHCP server, clients always use the router as DNS and AdGuard is bypassed entirely. Making AdGuard the DHCP server is the only way to advertise it as DNS directly in the lease. See [Architecture](../architecture.md).

## Resolved Issues

**AdGuard web UI unreachable after setup wizard**
The setup wizard moved the web UI from port 3000 to port 80, but port 80 was not in the Compose `ports:` mapping. Fixed by adding `80:80`. Later made irrelevant by switching to `network_mode: host`.

**LAN clients not receiving DHCP leases from AdGuard**
DHCP broadcasts from clients never reached the AdGuard container running on Docker bridge networking. Switching to `network_mode: host` resolved it — see [Why host networking](#why-host-networking-is-required-for-dhcp) above.

**AdGuard crash-looping after adding Nginx Proxy Manager**
NPM binds to port 80 on the host. AdGuard's web UI was also on port 80. Since AdGuard uses `network_mode: host`, both services competed for the same port. Fixed by changing AdGuard's web UI port to `3000` in `adguard/conf/AdGuardHome.yaml` under the `http.address` field before starting NPM.

**VM DNS queries failing**
DNS queries on the VM itself timed out when the nameserver was set to `127.0.0.1`. Docker port mappings bind to `0.0.0.0`, not the loopback interface. Fixed by setting the VM's nameserver to `192.168.2.3` instead.

**Mobile devices not resolving local domains**
`*.zabeu.dev` domains resolved on laptops but not on Android phones, with no queries appearing in AdGuard's query log. Android's Private DNS setting (Settings → Network & Internet → Private DNS) defaults to DNS-over-TLS with a hardcoded provider (e.g. `dns.google`), which bypasses whatever DNS AdGuard pushes via DHCP. Fix: set Private DNS to **Off** on the device. iOS equivalent: check Settings → Wi-Fi → [network name] → DNS for any manual override.
