# Networking

## Overview

| Layer | Public tier | Private tier |
|---|---|---|
| DNS | Proxied CNAME to the tunnel, one per app | DNS-only wildcard A record to the Tailscale IP |
| Hostname | `homelab-<app>.ashiqabdulkhader.dev` | `<app>.lab.ashiqabdulkhader.dev` |
| TLS | Cloudflare edge (Universal SSL) | Traefik, Let's Encrypt wildcard from cert-manager via DNS-01 |
| Transport to host | Cloudflare Tunnel | Tailscale (WireGuard) |
| Listener | Caddy (Docker) on `127.0.0.1:80`, plain HTTP | Traefik (k3s) hostPort on Tailscale IP `:443`, HTTPS |

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
- Private apps run in k3s behind Traefik. Traefik's pod port is published
  on the host as a `hostPort` with `hostIP` set to the node's Tailscale
  address, so the host forwards only `<Tailscale IP>:443` to it. They are
  not exposed on the LAN, on loopback to the tunnel, or on the internet.
- The Helm chart's own `hostIP` setting would also make Traefik *listen* on
  that address inside its pod, which fails, so the `hostIP` is added with a
  Flux post-render patch instead.
- **DNS:** a public, DNS-only (grey-cloud) wildcard record
  `*.lab.ashiqabdulkhader.dev` points at the Tailscale IP. Every tailnet
  device resolves it with ordinary DNS, so no split-DNS setup is needed. Off
  the tailnet the address cannot be routed.
- **TLS:** browsers require HTTPS for `.dev` (the whole TLD is on the HSTS
  preload list). Let's Encrypt cannot reach a Tailscale-only host for an
  HTTP-01 challenge, so cert-manager uses the **DNS-01** challenge through
  the Cloudflare API with a token limited to DNS edits on this one zone. The
  result is a real, publicly trusted wildcard certificate for
  `*.lab.ashiqabdulkhader.dev`, which Traefik serves as its default
  certificate.

### Boot ordering

A `hostPort` is implemented with iptables DNAT rules, not a listening
socket, so k3s does not need the Tailscale address to exist when it starts.
(The Docker-era settings remain: `net.ipv4.ip_nonlocal_bind = 1` and a
drop-in ordering `docker.service` after `tailscaled.service`.)

## Real client IPs

| Tier | Source |
|---|---|
| Public | `Cf-Connecting-Ip`, trusted only from private ranges (cloudflared reaches Caddy via the Docker bridge) |
| Private | The TCP source address, which is the client's Tailscale IP |

## Other DNS on the domain

The apex, `www`, the blog, mail and verification records on
`ashiqabdulkhader.dev` are unrelated to the homelab and are left untouched.
Homelab records carry the comment `home-lab tunnel -> Caddy`.
