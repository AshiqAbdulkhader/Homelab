# Decisions

Short records of choices that shape the homelab, newest first.

---

### 016: Open WebUI on NVIDIA NIM instead of a local model

**Context:** The local LLM was removed (decision 015) because a CPU model on
this host was too slow. NVIDIA's hosted catalog has a free tier with an
OpenAI-compatible API.

**Decision:** Open WebUI at `chat.lab`, pointed at
`integrate.api.nvidia.com/v1`, with the model list limited to the models that
answered a test request. Free-tier image and video generation is not wired in.

**Reason:** Far larger models than this box can run, at no hardware cost. The
trade-off is that prompts leave the network, the free tier is rate-limited, and
the catalog changes: 85 models were listed, 14 answered quickly, 4 queued for
minutes and about 45 returned 404. Re-probe before trusting the list.

---

### 015: Remove the AI stack (n8n, LLM, Open WebUI, SearXNG)

**Context:** The local LLM, Open WebUI, SearXNG and n8n were added in October
2026 (decisions 013 and 014). The host's memory bandwidth kept the model slow
and the apps were not being used.

**Decision:** Remove all four from `homelab-gitops`, along with their backup
hooks and the Local LLM page. Decisions 013 and 014 stay below as history.

**Reason:** They used RAM and backup effort for no benefit. Git still has the
manifests if they are wanted again.

---

### 014: Keep the CPU governor on powersave

**Context:** Local LLM speed is limited by memory bandwidth, so the
`performance` governor looked like a cheap win.

**Decision:** Leave it on `powersave`.

**Reason:** Measured on 3.6K-token prompts: prompt processing 102 to 107 tok/s
(powersave) against 105 to 109 (performance); generation 6.7 to 6.9 tok/s both
ways. The CPU's energy preference is already `performance`, so cores boost
freely. It would cost idle power for no gain. The real lever is the RAM (see decision 015).

---

### 013: A small CPU model (Qwen3-1.7B) behind llama.cpp

**Context:** A local model for n8n and chat, on a CPU-only host.

**Options:** Qwen3.5-4B (about 6 tok/s here), Qwen3-1.7B (about 17 to 24 tok/s),
sub-1B models (not reliable beyond extraction), Ollama instead of llama.cpp.

**Decision:** Qwen3-1.7B Q4_K_M on llama.cpp server, thinking off, one slot,
model pinned in RAM. Open WebUI as the UI, SearXNG for search.

**Reason:** Generation is memory-bandwidth bound, so the model size sets the
speed. 1.7B is the smallest model that is useful for classification, routing,
extraction and short answers at an interactive speed. llama.cpp gives direct
control of threads, cache and memory locking.

---

### 012: Recover from power loss and hangs without anyone present

**Context:** Three of the last four boots ended without a clean shutdown, and
the host stayed off for up to 2.7 days. There is no UPS.

**Decision:** Hardware watchdog (iTCO) fed by systemd, panic-on-lockup, a
network watchdog for the USB Wi-Fi, explicit boot ordering, and a boot report.
The BIOS must be set to power on after power loss. Details in
[Resilience](resilience.md).

**Reason:** The software already recovered by itself after a reboot; the gaps
were machines that stayed off and hangs nobody noticed.

---

### 011: A drop-only nftables table instead of ufw

**Context:** k3s exposes its API, the kubelet, node-exporter and flannel
VXLAN on every interface. ufw was installed but inactive.

**Options:** enable ufw with default deny (has to allow everything Docker
and k3s need, and is known to fight their rules); **a separate nftables
table that only drops the specific ports from untrusted interfaces**.

**Decision:** The nftables table. It cannot break anything it does not name,
and it is one small file in git.

**Consequences:** New LAN-facing ports are not blocked automatically; they
must be added to the set. Wazuh was deferred at the same time: too heavy
for one 22 GB host alongside the observability stack.

---

### 010: Secrets in git with SOPS and age

**Context:** With GitOps, the cluster's desired state, secrets included,
has to come from git.

**Options:** Sealed Secrets (cluster-held key, re-seal on rebuild);
External Secrets with a vault (another service to run); **SOPS + age**.

**Decision:** SOPS with an age key. Flux decrypts natively; only values are
encrypted, so diffs stay readable.

**Consequences:** The age key must be backed up and kept in a password
manager. Losing it means re-creating every secret.

---

### 009: Traefik on a hostPort bound to the Tailscale IP

**Context:** The private tier must stay unreachable from the LAN and the
tunnel, exactly as with Caddy.

**Options:** k3s ServiceLB (binds every node address); the Tailscale
Kubernetes operator (one tailnet device and `ts.net` name per app);
**Traefik with a `hostPort` whose `hostIP` is the Tailscale address**.

**Decision:** The last option. Same hostnames, same wildcard DNS record,
same security model; all routing is in Ingress manifests.

**Consequences:** The Tailscale IP appears in the Traefik HelmRelease.
Only one Traefik replica can hold the port, so updates briefly drop
connections.

---

### 008: Flux instead of Argo CD

**Context:** One controller should reconcile the cluster from GitHub.

**Decision:** Flux. Lighter (no UI server, a few hundred MB less memory),
everything is a CRD in git, native SOPS decryption, and bootstrap creates
the deploy key.

**Consequences:** No built-in web UI. Status comes from the `flux` CLI and
Grafana.

---

### 007: Move the private tier to k3s, managed by GitOps

**Context:** Docker Compose plus a hand-edited Caddyfile was fine for a few
services, but adding observability and more apps wants declarative,
reviewable, self-healing configuration.

**Decision:** Single-node k3s. Everything in it is declared in the private
`homelab-gitops` repository and applied by Flux. The public tier (one
Caddy on loopback for the tunnel) stays on Docker for now.

**Consequences:** One more layer (Kubernetes) to understand. Rebuilding the
cluster is `install k3s` plus `flux bootstrap`.

---

### 006: Private by default; public by exception

**Context:** Most self-hosted apps (dashboards, admin panels, media,
automation) are only for me and my devices, which are already on Tailscale.

**Decision:** Every service is reachable only over the tailnet unless there
is a specific reason to publish it. Public apps go through Cloudflare, and
those with logins get Cloudflare Access in front.

**Consequences:** Far less of the homelab faces the internet. Using a
private app needs Tailscale on the client device.

---

### 005: Private hostnames in public DNS, pointing at the Tailscale IP

**Context:** Private apps need names and trusted HTTPS (`.dev` forces HTTPS).

**Options:**
- MagicDNS name (`home-lab.<tailnet>.ts.net`) with Tailscale certificates.
  Only one hostname per node, so apps would need paths or ports.
- Split DNS on the tailnet for a private zone. More moving parts.
- **Public DNS-only wildcard `*.lab.<domain>` → Tailscale IP, with a Let's
  Encrypt wildcard certificate via DNS-01.**

**Decision:** The last option. One subdomain per app, works on every device
with no client configuration, and the address is unreachable off the tailnet.

**Consequences:** Needs a Cloudflare API token limited to DNS edits on the
zone, stored on the host. The Tailscale IP appears in public DNS, which
reveals nothing usable.

---

### 004: Flat `homelab-<app>` names for public services

**Context:** Free Universal SSL covers only one subdomain level.

**Decision:** Use `homelab-<app>.ashiqabdulkhader.dev` instead of
`<app>.homelab.ashiqabdulkhader.dev`.

**Consequences:** No paid certificate. Each public app needs its own DNS
record, created by a helper script.

---

### 003: Cloudflare Tunnel with a single catch-all rule to Caddy

**Context:** No port forwarding wanted; routing should live in one place.

**Decision:** The tunnel forwards every hostname to Caddy on loopback. Caddy
routes by hostname and returns 404 for unknown names.

**Consequences:** Publishing an app never needs a tunnel change, only DNS
and Caddy. Caddy is published on `127.0.0.1` so the LAN cannot bypass the
tunnel.

---

### 002: Caddy as the reverse proxy

**Context:** One proxy for both tiers, simple configuration, automatic
certificates.

**Decision:** Caddy, configured with a Caddyfile, plus the Cloudflare DNS
module for DNS-01.

**Alternatives:** Traefik (label-driven, more indirection), Nginx Proxy
Manager (UI-driven, harder to keep in git).

---

### 001: Docker CE instead of the Docker snap

**Context:** Ubuntu offered Docker as a snap.

**Decision:** Docker CE from Docker's official apt repository.

**Reason:** The snap is confined to `$HOME` and some removable-media paths,
which limits bind mounts and storage on other disks. It also differs from
upstream documentation. Docker CE is the standard setup.
