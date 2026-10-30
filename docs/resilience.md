# Resilience

Goal: after a power cut, a hang or a network drop, everything comes back
without anyone touching the machine. There is no UPS, so power loss is an
unclean shutdown; everything below assumes that.

## Failure modes and what handles them

| Failure | Handled by |
|---|---|
| Power cut, box stays off | BIOS **After Power Loss = Power On** (manual, once) |
| Kernel/PID 1 hang | Hardware watchdog (iTCO) fed by systemd; resets the host in about 30 s |
| Kernel lockup or oops | `panic_on_oops`, soft/hard-lockup panic, `kernel.panic=10`: reboot instead of hanging |
| Wi-Fi link dead | Network watchdog (every minute): reconnect, restart NetworkManager, reload the driver. NetworkManager retries forever |
| Tailscale stuck | Same watchdog restarts `tailscaled` after 5 bad checks |
| k3s starts before the Tailscale IP exists | `k3s` ordered after `tailscaled` and the firewall |
| Tunnel process exits | `cloudflared` has `Restart=always` and no start limit |
| Pod wedged | Liveness probes on every app that serves HTTP |
| Node under memory pressure | Critical pods run as `system-cluster-critical` (Traefik, cert-manager, Kuma, Prometheus, Grafana, Loki, Homepage) |
| Nobody knows it rebooted | Boot report: waits for a healthy stack, logs and notifies how long it took |
| Whole host down | External dead-man's switch (optional, see below); Uptime Kuma lives on the host and cannot report its own outage |

## Boot order

`network-online` → `tailscaled` → `homelab-firewall` → `k3s` (Flux then
reconciles everything else) and `docker` → Caddy → `cloudflared`.
`homelab-boot-report` runs last, once all of that is healthy.

Measured: about 38 s to userspace; the stack was back without help after the
reboot on 19 Oct.

## Boot report and the external check

`/etc/homelab/notify.env` (private, root only) takes:

- `TELEGRAM_BOT_TOKEN` / `TELEGRAM_CHAT_ID`: a message at each boot, with how
  long recovery took and whether the previous boot ended cleanly.
- `EXTERNAL_PING_URL`: a dead-man's-switch URL (e.g. a healthchecks.io
  check). Pinged every minute while the network is healthy; the service alerts
  when pings stop.

The boot log is also kept in `/var/log/homelab-boot.log`.

## Install and verify

```bash
sudo scripts/harden-boot.sh        # idempotent
journalctl -t homelab-net-watchdog -t homelab-boot --since today
cat /var/log/homelab-boot.log
systemctl show -p RuntimeWatchdogUSec
```

The scripts live in `homelab-config/host/resilience/`. Ordering changes and
the boot report take effect at the next start of the unit or the host.

## Status

Applied on the host (24 Oct): watchdog device present (`/dev/watchdog`, 30 s),
panic sysctls, k3s and cloudflared drop-ins, Wi-Fi retries forever, network
watchdog timer running, boot report enabled. **Not yet tested through a real
reboot or a forced hang**, and the BIOS "After Power Loss" setting is still to
be changed by hand. The notification credentials in `/etc/homelab/notify.env`
are still empty, so the boot report only logs.

## Not covered

- No UPS: a power cut is still an unclean shutdown (ext4 journal, SQLite and
  etcd recover, but in-flight writes are lost).
- The router or ISP being down: nothing on the host can fix that.
