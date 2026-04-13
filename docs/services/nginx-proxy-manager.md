# Nginx Proxy Manager

Runs as a Docker container on the Ubuntu VM. Acts as the single entry point for all local services — receives HTTPS traffic on port 443, terminates SSL, and forwards requests to the correct backend based on the domain name.

## Docker Compose

```yaml
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - '80:80'
      - '443:443'
      - '81:81'
    environment:
      TZ: 'America/Toronto'
    volumes:
      - ./npm/data:/data
      - ./npm/letsencrypt:/etc/letsencrypt
```

## SSL Certificate

Wildcard certificate covering `*.zabeu.dev` and `zabeu.dev`, issued by Let's Encrypt via DNS challenge. No ports are exposed to the internet — domain ownership is proven by creating a temporary TXT record in Cloudflare DNS.

| Setting | Value |
|---|---|
| DNS Provider | Cloudflare |
| Method | DNS challenge |
| API Token | Cloudflare token scoped to Edit DNS for `zabeu.dev` only |
| Key Type | ECDSA 256 |
| Renewal | Automatic |

## How Routing Works

AdGuard DNS rewrites resolve all `*.zabeu.dev` subdomains to `192.168.2.3` (the VM). NPM receives the request on port 443, reads the domain name, and forwards to the correct backend. SSL is terminated at NPM — backends run plain HTTP internally, except Proxmox which runs its own HTTPS.

## Proxy Hosts

| Domain | Backend IP | Port | Scheme | WebSockets |
|---|---|---|---|---|
| `proxmox.zabeu.dev` | `192.168.2.2` | `8006` | https | ✅ |
| `adguard.zabeu.dev` | `192.168.2.3` | `3000` | http | ❌ |
| `npm.zabeu.dev` | `192.168.2.3` | `81` | http | ❌ |

## Resolved Issues

**Port 80 conflict with AdGuard**
NPM binds to port 80 on the host. AdGuard's web UI was also configured for port 80, causing AdGuard to crash-loop on startup. Fixed by moving AdGuard's web UI to port 3000 in `AdGuardHome.yaml` before starting NPM. See [AdGuard resolved issues](adguard.md#resolved-issues).
