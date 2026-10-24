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
| n8n | `n8n.lab.ashiqabdulkhader.dev` | Workflow automation | Owner account set at first visit; SQLite on a `local-path` volume; encryption key in a SOPS secret; tailnet only, so inbound webhooks from the internet do not reach it |
| Open WebUI | `chat.lab.ashiqabdulkhader.dev` | Chat UI for the local LLM, with web search | First account is admin, later sign-ups need approval; title/tag generation off to save the slow model |
| Local LLM | `llm.lab.ashiqabdulkhader.dev/v1`, in-cluster `llm.llm.svc:8080` | OpenAI-compatible API (llama.cpp, Qwen3-1.7B Q4_K_M, CPU) | API key (SOPS); about 20 tok/s generation, about 120 tok/s prompt processing; model pinned in RAM |
| SearXNG | `search.lab.ashiqabdulkhader.dev`, in-cluster `searxng.searxng.svc:8080` | Metasearch with JSON output for Open WebUI and n8n | No login; tailnet only |
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
| Docker CE | Container runtime for the public-tier Caddy (stock `caddy:2` image) |
| homelab-backup (systemd timer) | Nightly restic backup to Cloudflare R2. See [Backups](backups.md) |
| homelab-firewall | nftables guard for k3s ports. See [Architecture](architecture.md#host-firewall) |

## Planned

| Service | Tier | Purpose |
|---|---|---|
| Cloudflare Access | Public (edge) | Identity check in front of public apps that have logins |

## Deferred

| Service | Why not now |
|---|---|
| Wazuh | Its indexer alone needs several GB of RAM; revisit if the host gets more memory or a second node |

## Adding a row

When a service is deployed, add it here with its tier and hostname. If it is
public, write down why it needs to be.
