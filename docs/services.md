# Services

Private (Tailscale-only) is the default tier. A service is listed as public
only when it is deliberately published to the internet.

## Private (k3s, managed by Flux)

| Service | Hostname | Purpose | Notes |
|---|---|---|---|
| Landing page | `lab.ashiqabdulkhader.dev` | Links to every private service | nginx serving one HTML file from a ConfigMap |
| Grafana | `grafana.lab.ashiqabdulkhader.dev` | Dashboards; logs via Loki | Admin password in a SOPS secret |
| Prometheus | `prometheus.lab.ashiqabdulkhader.dev` | Metrics, 15 days | No login; tailnet only |
| Traefik dashboard | `traefik.lab.ashiqabdulkhader.dev/dashboard/` | Routers and services | Read-only |
| Uptime Kuma | `kuma.lab.ashiqabdulkhader.dev` | Uptime checks, backup heartbeats, Telegram alerts | See [Monitoring](monitoring.md) |
| whoami | `whoami.lab.ashiqabdulkhader.dev` | Echoes request headers to test routing | |

### Cluster infrastructure (no UI)

| Component | Purpose |
|---|---|
| Flux | Applies `homelab-gitops` from GitHub |
| Traefik | Ingress controller on the Tailscale IP |
| cert-manager | `*.lab` wildcard certificate from Let's Encrypt (DNS-01) |
| Loki | Log storage, 14 days |
| Alloy | Ships pod logs and the host's systemd journal to Loki |
| node-exporter, kube-state-metrics | Host and cluster metrics |

## Public (Docker)

| Service | Hostname | Purpose | Notes |
|---|---|---|---|
| Landing page | `homelab.ashiqabdulkhader.dev` | Shows that the public path works | Served by Caddy directly |

## Host services

| Service | Purpose |
|---|---|
| k3s | Kubernetes for every private app |
| cloudflared | Cloudflare Tunnel connector for the public tier |
| tailscaled | Tailnet membership, private tier, Tailscale SSH |
| Docker CE | Container runtime for the public-tier Caddy |
| homelab-backup (systemd timer) | Nightly restic backup to Cloudflare R2. See [Backups](backups.md) |

## Planned

| Service | Tier | Purpose |
|---|---|---|
| Cloudflare Access | Public (edge) | Identity check in front of public apps that have logins |
| Wazuh | Private | Security monitoring (SIEM, host intrusion detection) |
| Host firewall | Host | Limit k3s API, kubelet and node-exporter to loopback and the tailnet |

## Adding a row

When a service is deployed, add it here with its tier and hostname. If it is
public, write down why it needs to be.
