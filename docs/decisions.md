# Decisions

Short records of choices that shape the homelab, newest first.

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
