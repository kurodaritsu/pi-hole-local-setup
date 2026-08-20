# Pi-hole Local Setup

Single-device local-only Pi-hole DNS through a container. This is the setup that I use at the moment.

Tested on `podman` (with `podman-compose`) - commands below use `docker`
syntax, swap in `podman-compose` if that's your stack.

Can't guarantee that this setup will work by default. See [Troubleshooting](./README.md#troubleshooting) for details.

> *Q: Why don't you apply this to your router so your other devices have pi-hole as well?*

I can do that. I just need to get access to the router settings which I don't have access at the moment (for reasons I don't want to get into). Can update this repo to a router-level config in the future.

## Requirements

[`docker compose`](https://docs.docker.com/engine/install/) (or [`podman-compose`](https://podman.io/docs/installation)) already installed on this host.

## How to run

1. Clone this repo
2. Create `.env` file. See [`.env.example`](./.env.example) for details
3. [Setup `resolv.conf` with your upstream nameservers](./README.md#set-upstream-nameservers)
4. Run `docker compose up -d` (or `podman-compose up -d`)
5. [Set Pi-hole container as device's DNS resolver](./README.md#set-pi-hole-container-as-devices-dns-resolver)

After which, your Pi-hole is now active to your device.

> [!NOTE]
> For official guides, see details in Pi-hole website: https://pi-hole.net/

## Web UI

After running the container, check:

http://127.0.0.1/admin

Password is whatever's set in `.env` (`ADMIN_PASSWORD`). If left unset,
`pihole/pihole:latest` generates a random password on first start - check it
with `docker compose logs pihole`.

## `systemd` service for auto-start on login

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

`compose` alone does not make your system use Pi-hole. Pick one whichever applies to you:

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
one per line:

```
nameserver 1.1.1.1
nameserver 1.0.0.1
```

The example above points to Cloudflare - swap in whatever upstream you
prefer (your ISP's resolver, Google's `8.8.8.8`/`8.8.4.4`, a self-hosted
resolver, etc). `compose.yaml` also sets `FTLCONF_dns_upstreams` to match;
update both if you change the nameservers.

This file is bind-mounted into the container to bypass podman's per-network DNS
forwarder (aardvark-dns) - without it, the container's own startup DNS
check hairpins back to itself and hangs.

If the host's `systemd-resolved` stub listener occupies port 53
(`127.0.0.53`/`127.0.0.54`), the container can't bind `0.0.0.0:53` at
all. Disable the stub listener (`DNSStubListener=no` in
`/etc/systemd/resolved.conf`) or free port 53 some other way before
starting the container.
