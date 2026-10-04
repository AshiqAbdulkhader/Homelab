# Operations

Commands run on the host from the private configuration repository.

## Stacks

```bash
scripts/stack.sh new <app>          # scaffold stacks/<app> and data/<app> from the template
scripts/stack.sh up <app|all>       # start (creates the shared `homelab` network if missing)
scripts/stack.sh down <app|all>
scripts/stack.sh pull all && scripts/stack.sh up all   # update images
scripts/stack.sh logs <app>
scripts/stack.sh ps all
```

Every stack:

- joins the external `homelab` network,
- publishes **no** host ports (only Caddy does),
- keeps persistent data in `data/<app>/`,
- keeps secrets in `stacks/<app>/.env`, which is not committed.

## Making a service reachable

### Private (default)

1. Deploy the stack on the `homelab` network.
2. Add a matcher inside the `*.lab` site block of the Caddyfile:
   ```
   @app host app.lab.{$DOMAIN}
   handle @app {
   	reverse_proxy app:8080
   }
   ```
3. Reload Caddy:
   ```bash
   docker exec caddy caddy reload --config /etc/caddy/Caddyfile
   ```

The wildcard DNS record and wildcard certificate already cover the new name.

### Public (opt-in)

1. Decide whether it really needs to be public. If it has a login, put
   Cloudflare Access in front of it.
2. Add a site block for the public listener:
   ```
   http://homelab-app.{$DOMAIN} {
   	import common
   	reverse_proxy app:8080
   }
   ```
3. Reload Caddy, then create the DNS record:
   ```bash
   scripts/expose.sh app            # proxied CNAME homelab-app -> tunnel
   scripts/expose.sh --remove app   # undo
   ```

## Checks

```bash
# Public path, from the host, without going through Cloudflare
curl -H 'Host: homelab-app.ashiqabdulkhader.dev' http://127.0.0.1/

# Tunnel picked up its configuration
journalctl -u cloudflared | grep 'Updated to new configuration'

# Private path, from any tailnet device
curl https://app.lab.ashiqabdulkhader.dev/
```

## Host setup notes

- Docker CE replaced the Ubuntu snap package. The snap version cannot read
  files outside `$HOME` and has packaging quirks.
- The admin user is in the `docker` group.
- `cloudflared` runs as a systemd unit with a root-only token file.
