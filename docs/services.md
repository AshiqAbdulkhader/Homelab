# Services

Private (Tailscale-only) is the default tier. A service is listed as public
only when it is deliberately published to the internet.

## Running

| Service | Tier | Hostname | Purpose | Notes |
|---|---|---|---|---|
| Caddy | Infrastructure | n/a | Reverse proxy for both tiers | Custom build with the Cloudflare DNS module for DNS-01 |
| Landing page | Public | `homelab.ashiqabdulkhader.dev` | Shows that the public path works | Served by Caddy directly |
| Private landing page | Private | `lab.ashiqabdulkhader.dev` | Shows that the private path works | Served by Caddy directly |
| whoami | Private | `whoami.lab.ashiqabdulkhader.dev` | Echoes request headers to test routing | Was public during setup; now private |
| Uptime Kuma | Private | `kuma.lab.ashiqabdulkhader.dev` | Uptime checks, backup heartbeats, Telegram alerts | See [Monitoring](monitoring.md) |

## Host services

| Service | Purpose |
|---|---|
| cloudflared | Cloudflare Tunnel connector for the public tier |
| tailscaled | Tailnet membership, private tier, Tailscale SSH |
| Docker CE | Container runtime |
| homelab-backup (systemd timer) | Nightly restic backup to Cloudflare R2. See [Backups](backups.md) |

## Planned

| Service | Tier | Purpose |
|---|---|---|
| Cloudflare Access | Public (edge) | Identity check in front of public apps that have logins |

## Adding a row

When a service is deployed, add it here with its tier and hostname. If it is
public, write down why it needs to be.
