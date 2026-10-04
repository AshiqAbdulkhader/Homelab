# Decisions

Short records of choices that shape the homelab, newest first.

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
