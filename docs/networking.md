# Networking

## Overview

| Layer | Public tier | Private tier |
|---|---|---|
| DNS | Proxied CNAME to the tunnel, one per app | DNS-only wildcard A record to the Tailscale IP |
| Hostname | `homelab-<app>.ashiqabdulkhader.dev` | `<app>.lab.ashiqabdulkhader.dev` |
| TLS | Cloudflare edge (Universal SSL) | Caddy, Let's Encrypt wildcard via DNS-01 |
| Transport to host | Cloudflare Tunnel | Tailscale (WireGuard) |
| Caddy listener | `127.0.0.1:80`, plain HTTP | Tailscale IP `:443`, HTTPS |

## Cloudflare Tunnel (public tier)

- A **remotely managed** tunnel named `Home-Lab`. Its ingress rules are stored
  in Cloudflare, and `cloudflared` on the host authenticates with a token
  file readable only by root.
- Ingress has a single catch-all rule, `→ http://localhost:80`. Routing by
  hostname happens in Caddy, so publishing an app needs only:
  1. a Caddy site block, and
  2. a proxied DNS CNAME `homelab-<app>` → `<tunnel-id>.cfargotunnel.com`.
- No router port forwarding, and the home IP address is never exposed.

### Why `homelab-<app>` and not `<app>.homelab`

Cloudflare's free Universal SSL certificate covers the apex and **one** level
of subdomain (`*.ashiqabdulkhader.dev`). A name like `app.homelab.ashiqabdulkhader.dev`
would need a paid Advanced Certificate. A flat, prefixed name keeps public
apps grouped and free.

## Tailscale (private tier)

- `home-lab` is a node on my tailnet with Tailscale SSH enabled.
- Private apps are served by Caddy on the node's Tailscale address only. They
  are not exposed on the LAN, on loopback to the tunnel, or on the internet.
- **DNS:** a public, DNS-only (grey-cloud) wildcard record
  `*.lab.ashiqabdulkhader.dev` points at the Tailscale IP. Every tailnet
  device resolves it with ordinary DNS, so no split-DNS setup is needed. Off
  the tailnet the address cannot be routed.
- **TLS:** browsers require HTTPS for `.dev` (the whole TLD is on the HSTS
  preload list). Let's Encrypt cannot reach a Tailscale-only host for an
  HTTP-01 challenge, so Caddy uses the **DNS-01** challenge through the
  Cloudflare API with a token limited to DNS edits on this one zone. The
  result is a real, publicly trusted wildcard certificate for
  `*.lab.ashiqabdulkhader.dev`.

### Boot ordering

Docker publishes the private listener on the Tailscale IP. If Docker started
before `tailscaled` had brought that address up, the bind would fail. Two
host settings prevent this:

- `net.ipv4.ip_nonlocal_bind = 1`, so a socket can bind an address that isn't
  present yet.
- A systemd drop-in that orders `docker.service` after `tailscaled.service`.

## Real client IPs

| Tier | Source |
|---|---|
| Public | `Cf-Connecting-Ip`, trusted only from private ranges (cloudflared reaches Caddy via the Docker bridge) |
| Private | The TCP source address, which is the client's Tailscale IP |

## Other DNS on the domain

The apex, `www`, the blog, mail and verification records on
`ashiqabdulkhader.dev` are unrelated to the homelab and are left untouched.
Homelab records carry the comment `home-lab tunnel -> Caddy`.
