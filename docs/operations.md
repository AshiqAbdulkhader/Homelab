# Operations

Private apps are changed through git: commit to `homelab-gitops`, and Flux
applies it. The host-level pieces are managed from `homelab-config`.

## GitOps workflow

```bash
# after pushing a change
flux reconcile kustomization flux-system --with-source   # apply now instead of waiting

# status
flux get kustomizations
flux get helmreleases -A
flux logs --level=error
kubectl get pods -A
kubectl get ingress -A
```

Order of application: `infra-controllers` (Traefik, cert-manager) →
`infra-configs` (issuer, wildcard certificate) → `monitoring` and `apps`.

## Adding a private service (default)

1. In `homelab-gitops`, create `apps/<app>/` with:
   - `kustomization.yaml` setting `namespace: <app>`
   - `namespace.yaml`
   - a Deployment, a Service, and an Ingress for `<app>.lab.ashiqabdulkhader.dev`
     with `ingressClassName: traefik`
2. Add the directory to `apps/kustomization.yaml`.
3. Commit and push.

The wildcard DNS record and wildcard certificate already cover the new name,
and Traefik serves the certificate by default, so the Ingress needs no
`tls:` section. Persistent data uses a `local-path` PVC.

Secrets go in a SOPS-encrypted `*.sops.yaml` file:

```bash
kubectl create secret generic <name> -n <app> --from-literal=key=value \
  --dry-run=client -o yaml > apps/<app>/<name>.sops.yaml
sops --encrypt --in-place apps/<app>/<name>.sops.yaml
```

Then add an Uptime Kuma monitor for the service.

## Publishing a service (opt-in)

Public apps are still served by Caddy (Docker) on the tunnel listener.

1. Decide whether it really needs to be public. If it has a login, put
   Cloudflare Access in front of it.
2. Add a site block for the public listener in `homelab-config`:
   ```
   http://homelab-app.{$DOMAIN} {
   	import common
   	reverse_proxy app:8080
   }
   ```
3. Reload Caddy, then create the DNS record:
   ```bash
   docker exec caddy caddy reload --config /etc/caddy/Caddyfile
   scripts/expose.sh app            # proxied CNAME homelab-app -> tunnel
   scripts/expose.sh --remove app   # undo
   ```

## Updates

- **Helm charts** are pinned to a minor version (for example `41.6.x`); Flux
  picks up patch releases on its own. For a minor or major bump, edit the
  `version` in the HelmRelease and push.
- **App images** are pinned to a tag in each Deployment. Bump the tag and push.
- **k3s:** rerun the install script with a newer `K3S_VERSION`.
- **Caddy** (public tier): `docker compose build --pull && docker compose up -d`
  in `stacks/caddy`.

## Checks

```bash
# Private path, from any tailnet device
curl https://whoami.lab.ashiqabdulkhader.dev/

# The LAN must NOT reach the private tier (expect connection refused)
curl -k https://<home-lab LAN IP>/

# Public path, from the host, without going through Cloudflare
curl -H 'Host: homelab.ashiqabdulkhader.dev' http://127.0.0.1/

# The tunnel must NOT serve private names (expect 404)
curl -H 'Host: whoami.lab.ashiqabdulkhader.dev' http://127.0.0.1/

# Wildcard certificate
kubectl -n traefik get certificate lab-wildcard
```

## Host setup notes

- k3s is installed by `scripts/install-k3s.sh` with Traefik and ServiceLB
  disabled and storage in `/srv/k3s/storage`.
- Docker CE replaced the Ubuntu snap package. The snap version cannot read
  files outside `$HOME` and has packaging quirks.
- `cloudflared` runs as a systemd unit with a root-only token file.
- The SOPS age key is at `~/.config/sops/age/keys.txt`, backed up nightly and
  kept in the password manager.
