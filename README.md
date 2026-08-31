# Pi-hole Local Setup

LAN-wide Pi-hole setup deployed to Raspberry Pi 3B+. This is the setup that I use at the moment.

Uses `docker` inside Pi as I'm also deploying other containers here as well.

Can't guarantee that this setup will work on your end by default. See [Troubleshooting](#troubleshooting) for details.

## Pre-requisites

- [`docker compose`](https://docs.docker.com/engine/install/) (or [`podman-compose`](https://podman.io/docs/installation)) already installed on this host.
- Your Raspberry Pi's LAN address (i.e. `<pi-hole-ip-address>`)
- Set your router's Primary DNS Server to `<pi-hole-ip-address>`

## How to run

1. From Raspberry Pi, clone this repo
2. Create `.env` file. See [`.env.example`](./.env.example) for details
3. [Setup your upstream nameservers](#set-upstream-nameservers)
4. Run `docker compose up -d` (or `podman-compose up -d`)

After which, your Pi-hole is now active to your device.

> [!NOTE]
> For official guides, see details in Pi-hole website: https://pi-hole.net/

## Web UI

After running the container, check:
```
http://<pi-hole-ip-address>/admin
```

Password is whatever's set in `.env` (`ADMIN_PASSWORD`). If left unset,
`pihole/pihole:latest` generates a random password on first start - check it
with `docker compose logs pihole`.

## `systemd` service for auto-start on login

> [!NOTE]
> This applies only to purely local setup. Some systems need to set this
> up manually because most of the time, docker does this for you
> automatically by: `restart: unless-stopped`

You can try running the container on startup so you have Pi-hole automatically.

1. Create `~/.config/systemd/user/pihole-compose.service` (swap `docker` for
`podman` and adjust `WorkingDirectory` if needed):

```conf
[Unit]
Description=Pi-hole compose stack
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=%h/path/to/pi-hole-local-setup
ExecStart=docker compose up -d
ExecStop=docker compose down

[Install]
WantedBy=default.target
```

2. Reload and enable it:

```
systemctl --user daemon-reload
systemctl --user enable --now pihole-compose.service
```

3. Check status:

```
systemctl --user status pihole-compose.service
```

If your `docker`/`podman` daemon itself needs a login session to run
(rootless setups), also enable lingering so the service starts even without
an active login:

```
loginctl enable-linger $USER
```

## Troubleshooting

> [!NOTE]
> For more information on Pi-hole setup, you can check the official Pi-hole forum: https://discourse.pi-hole.net/

### Set Pi-hole container as device's DNS resolver

For local setup, `compose` alone does not make your system use Pi-hole. Pick one whichever applies to you:

**a) Persistent, via NetworkManager:**

```
nmcli connection modify <conn-name> ipv4.dns "127.0.0.1" ipv4.ignore-auto-dns yes
nmcli connection up <conn-name>
```

**b) Quick test, via resolvectl (does not persist across reboot):**

```
resolvectl dns <iface> 127.0.0.1
resolvectl domain <iface> "~."
```

### Set upstream nameservers

Pi-hole needs somewhere to forward queries it doesn't block. Edit
[`resolv.conf`](./resolv.conf) and list the nameserver(s) you want to use,
one per line.

> [!WARNING]
> Keep `resolv.conf` present in the repo root. If it's deleted, `podman`
> silently creates a *directory* at that bind-mount path instead of
> erroring, and the container fails in a confusing way.

By default, it points to Cloudflare (ipv4 and ipv6). Change whatever
upstream you prefer. [`compose.yaml`](./compose.yaml) also sets
`FTLCONF_dns_upstreams` to match. Update both if you change the nameservers.

The [`resolv.conf`](./resolv.conf) is bind-mounted into the container
to bypass podman's per-network DNS forwarder (aardvark-dns). Without it,
the container's own startup DNS check hairpins back to itself and hangs.

If the host's `systemd-resolved` stub listener occupies port 53
(`127.0.0.53`/`127.0.0.54`), the container can't bind `0.0.0.0:53` at
all. Disable the stub listener (`DNSStubListener=no` in
`/etc/systemd/resolved.conf`) or free port 53 some other way before
starting the container.

### Other caveats

Rootless podman binding port 53 requires lowering the host's
unprivileged port floor:

```
net.ipv4.ip_unprivileged_port_start=53
```

This setup is host-wide and persists across reboots (see `/etc/sysctl.d/`).
It means *any* unprivileged local process on this machine can now bind
ports 53-1023, not just this container. Accepted tradeoff for running
Pi-hole rootless on port 53 - only a concern if you run other, less
trusted workloads on the same host.
