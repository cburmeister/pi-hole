pi-hole
====

Pi-hole blocks advertisements for all devices on the network. Unbound resolves
DNS queries recursively.

Pi-hole receives a query and applies its block lists. It then sends the query to
unbound. Unbound asks the root servers directly. It does not use a public
resolver, so no single company sees all of the queries.

```
device -> router -> pihole :53 (filter) -> unbound :5053 (recursive)
```

The stack uses two images. Docker pulls each image by digest. There is no build
step and no submodule.

- [pihole/pihole](https://hub.docker.com/r/pihole/pihole), from the Pi-hole project
- [klutchell/unbound](https://github.com/klutchell/unbound-docker)

## How to deploy

Copy the repository to the device:

```bash
rsync -av --exclude '.git' ./ <user>@<pi>:/opt/pi-hole/
```

Start the stack:

```bash
ssh <user>@<pi>
cd /opt/pi-hole
docker compose up -d
```

To upgrade, merge the Dependabot pull request. The pull request changes an image
digest. Then do the two steps above again.

## Configuration

Copy `.env.example` to `.env`. Put the file in the same directory as
`docker-compose.yaml`. Git ignores `.env`.

Each value is necessary. If a value is missing, Docker Compose does not start the
stack. It gives you the name of the missing value.

| Variable | Description |
|---|---|
| `FTLCONF_webserver_api_password` | The password for the Pi-hole admin page |
| `FTLCONF_dns_upstreams` | The resolver that Pi-hole sends queries to. Use `127.0.0.1#5053` |
| `TZ` | The time zone |
| `FTLCONF_misc_dnsmasq_lines` | dnsmasq directives, semicolon separated. Holds `edns-packet-max` and any local DNS records |

### Local DNS records

Put local names in `FTLCONF_misc_dnsmasq_lines`. One `address=` directive covers
a subdomain and everything below it:

```
edns-packet-max=1232;address=/home.example.com/192.168.1.100
```

That makes `home.example.com` and every name under it resolve to one address.
This is sufficient to send all internal services to a single reverse proxy. No
record per service is necessary.

Keep `edns-packet-max` first. Separate each directive with a semicolon.

### Pi-hole v6 uses different variable names

Pi-hole v5 used these names: `WEBPASSWORD`, `DNS1`, `DNS2`, `INTERFACE` and
`DNSMASQ_LISTENING`.

Pi-hole v6 does not know these names. If you give v6 a name that it does not
know, it does not show an error. It ignores the name. It then uses the default
value from the image.

A configuration that uses v5 names looks correct. But Pi-hole sends its queries
to a public resolver, and unbound receives no queries.

### unbound

The unbound image is
[distroless](https://github.com/GoogleContainerTools/distroless). It has no shell
and no package manager. Its entrypoint is the unbound program.

The image supplies `/etc/unbound/unbound.conf`. That file is sufficient for this
stack. It gives you DNSSEC validation, RFC1918 access control, RFC1918 private
addresses, an EDNS buffer size of 1232 bytes, prefetching and cache sizing. The
image also supplies `root.key` and `root.hints` in `/var/unbound`.

This repository therefore contains no full unbound configuration.

To add configuration, put a file in `/etc/unbound/custom.conf.d/`. The supplied
configuration reads that directory with an `include-toplevel` statement. Each
file must be readable by user 101 and group 102, or by all users.

`unbound/hardening.conf` uses this method. It sets six options that the image
does not set. Two of them are important: `harden-algo-downgrade` and
`unwanted-reply-threshold`.

Do not replace `/etc/unbound/unbound.conf` with a bind mount. This removes DNSSEC
validation, the private address rules and the cache settings.

Do not put a volume on `/var/unbound`. The volume hides `root.key` and
`root.hints`. DNSSEC validation then stops.

## How to verify

Do not use a successful lookup as proof. A lookup is successful when Pi-hole
sends its queries to a public resolver and unbound receives nothing.

A `SERVFAIL` result for a bad signature is also insufficient proof. Public
resolvers do DNSSEC validation too. The result is the same in both conditions.

### 1. Make sure that Pi-hole sends its queries to unbound

This is the necessary test. Do it first.

```bash
cd /opt/pi-hole
docker compose exec pihole grep -a forwarded /var/log/pihole/pihole.log | tail
```

Each line must contain `127.0.0.1#5053`. If a line contains a public resolver,
the value of `FTLCONF_dns_upstreams` is wrong.

This command reads a file. Do not use `docker compose logs`. Pi-hole writes each
query to `/var/log/pihole/pihole.log`, which is a tmpfs. It writes only its start
messages to stdout.

### 2. Make sure that unbound does the DNSSEC validation

```bash
docker compose exec pihole dig +ad dnssec.works @127.0.0.1 -p 5053
```

The result must be `NOERROR`. The flags must contain `ad`.

Raspberry Pi OS Lite has no `dig` program. The Pi-hole image has one. Pi-hole
uses host networking, so `127.0.0.1:5053` in the container is the loopback
address of the host. Unbound listens there.

### 3. Make sure that unbound refuses a bad signature

```bash
dig @<pi> fail01.dnssec.works
```

The result must be `SERVFAIL`. The result must contain no address.

### 4. Look at the admin page

Open Settings, then DNS. The page must show a custom resolver of
`127.0.0.1#5053`. The boxes for Cloudflare, Google and Quad9 must be empty.

## Notes

Pi-hole does not set the `ad` flag. Its `dns.dnssec` option is false, and it must
stay false. Unbound already did the validation. For this reason, test 2 above
sends its query to unbound on port 5053. It does not use Pi-hole on port 53.

Pi-hole uses host networking. This lets it use port 53 on the LAN.

Unbound listens on port 53 in its container. Docker publishes that port to
`127.0.0.1:5053` on the host. No other device can reach unbound.

Pi-hole starts after unbound becomes healthy. Docker Compose does this with
`depends_on` and `condition: service_healthy`. The health check uses `drill-hc`,
a program in the unbound image, to resolve `dnssec.works`.

The health check shows less than it appears to. `drill-hc` exits 0 for any
response, and `SERVFAIL` is a response. unbound therefore reports `healthy` while
it is answering but cannot resolve anything, for example if outbound port 53 is
blocked. The check proves that the daemon runs. It does not prove that resolution
works. Use the steps in **How to verify** for that.

Make sure that no other program uses port 53 before the first start:

```bash
sudo ss -lunp | grep ':53'
```

Avahi uses port 5353 for mDNS. This is not a conflict. `systemd-resolved` uses
port 53. Stop it if it is present.

A reboot takes two or three minutes. Unbound must pass its health check before
Pi-hole starts. The stack looks unavailable during this time.

Pi-hole shows this message at the first start: `No database file found, creating
new (empty) database`. This is normal.

`SYS_NICE` lets Pi-hole increase its process priority. Pi-hole gives a warning at
each start without it. `SYS_TIME` is absent on purpose. That capability lets the
container set the clock of the host. Raspberry Pi OS uses `systemd-timesyncd` for
the clock. The `FTLCONF_ntp_sync_active` variable stops the NTP client of
Pi-hole.

### Writes to the SD card

The stack keeps writes to the SD card low.

`FTLCONF_database_maxDBdays` is `0`. This stops the long-term query database,
which does the most writes. Pi-hole keeps 24 hours of history in memory, so the
dashboard continues to operate. The history does not survive a restart. Set a
positive value to keep a permanent history.

`/var/log/pihole` is a tmpfs. It still supplies the `forwarded` lines that test 1
uses.

Pi-hole writes `gravity.db` only when it updates the block lists.
