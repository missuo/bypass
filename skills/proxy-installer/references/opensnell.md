# OpenSnell (`snell-server`)

Snell v4/v5 proxy server installer. One-liner: `bash <(curl -fsSL https://s.ee/opensnell)`
Source: https://github.com/missuo/opensnell · script `install.sh`

- **Linux + systemd only.** Depends on systemd, UFW/firewalld, and `/etc/sysctl.conf`. On macOS/Windows/BSD build from source: `go install github.com/missuo/opensnell/cmd/snell-server@latest`.
- **Root required.**
- **Architectures:** OpenSnell variant → amd64 / arm64 / 386 / armv7. Surge variants → amd64 / aarch64 / i386 / armv7l (v6 drops armv7l).

## Paths

| Item | Path |
|------|------|
| Binary | `/usr/local/bin/snell-server` |
| Config | `/etc/snell/snell-server.conf` |
| Install metadata | `/etc/snell/.install_meta` (variant, version, port, PSK, etc.) |
| systemd unit | `/etc/systemd/system/snell-server.service` |
| Service name | `snell-server` |

## Subcommands

`install` · `reconfigure` · `update` (alias `upgrade`) · `uninstall` · `start` · `stop` · `restart` · `enable` · `disable` · `status` · `info` · `help`. No argument → interactive menu (options 0–11).

## Dependencies

`ensure_tools` installs, via apt/dnf/yum: `curl`, `unzip` (only needed to unpack the Surge zip), `openssl`, `ss` (from `iproute2`). On other package managers install these manually first (see SKILL.md "Dependency handling").

## Install variants (first prompt: `Variant [1]`)

1. **OpenSnell** (default) — self-hosted, GPLv3, all architectures. Recommended.
2. **Surge official snell-server v5.0.1** — closed-source, Linux only.
3. **Surge official snell-server v6.0.0b3** — closed-source beta. Adds the `mode` directive, statically linked, multi-core. No armv7l. Extra confirm prompt before install.

## Config prompts (in order) — for scripting a heredoc

For the **default OpenSnell variant** (`install` with variant 1), the prompt order is:

1. `Variant [1]:` → `1`
2. `Listen port [<random>]:` → blank = random free port (10000–60000)
3. `PSK / shared secret [] :` → blank = auto-generate 32-char PSK
4. `obfs mode (off/http/tls) [off]:` → `off` / `http` / `tls`
5. `Allow IPv6 destinations (y/n) [y]:`
6. `Accept UDP-over-TCP (y/n) [y]:`
7. `Enable QUIC proxy mode (y/n) [y]:` (required for Surge HTTP/3)
8. `Egress interface [] :` → blank = default route
9. `Enable TCP Fast Open (y/n) [n]:`

`</dev/null` accepts all of the above and yields a working default server.

**Surge v6 (variant 3)** replaces some prompts: after PSK it asks `Traffic mode (default/unshaped/unsafe-raw) [default]`, then `DNS IP preference (default/prefer-ipv4/prefer-ipv6/ipv4-only/ipv6-only)` and custom DNS servers, instead of `obfs`/`ipv6`. It also asks for an egress interface and defaults TFO to `y`.

### Key constraints
- **v6 PSK must be 16–255 bytes** or the server refuses to start (the script validates this).
- **v6 `mode` must match on the client.** `default` = obfs + AES; `unshaped` = AES only (~10% faster, wire looks like Snell v3); `unsafe-raw` = plaintext, no crypto (trusted intranets/tunnels only).
- Enabling TFO writes `net.ipv4.tcp_fastopen = 3` to `/etc/sysctl.conf` and applies it.

## Resulting config file (v5, OpenSnell variant)

```ini
[snell-server]
listen = 0.0.0.0:<port>
psk = <psk>
obfs = off
ipv6 = true
udp = true
quic = true
egress-interface =
tfo = false
```

## Firewall

Auto-opens the port on UFW or firewalld (TCP, plus UDP when `quic = true`). Also open it in the cloud provider's security group if applicable.

## Output / client share formats (`info` reprints these)

- Node name with geo flag emoji (e.g. `🇯🇵 JP A1B2`), resolved via ip.sb + ipinfo.
- **Surge** `[Proxy]` line: `<name> = snell, <ip>, <port>, psk="<psk>", version=<proto>[, tfo=true]`
- **Mihomo/Clash.Meta** proxy: `- {name: ..., server: ..., port: ..., type: snell, psk: ..., version: <proto>}`
- **Shadowrocket** URL: `snell://<base64>?tfo=<0|1>&version=<proto>#<name>`

## Common operations

```bash
curl -fsSL https://s.ee/opensnell -o /tmp/opensnell.sh
sudo bash /tmp/opensnell.sh install </dev/null   # default install, non-interactive
sudo bash /tmp/opensnell.sh info                 # print IP/port/PSK + all share lines
sudo bash /tmp/opensnell.sh reconfigure          # rewrite config, restart
sudo bash /tmp/opensnell.sh update               # re-download binary (same variant/major)
sudo bash /tmp/opensnell.sh uninstall            # remove binary + unit (asks about config)
systemctl is-active snell-server
journalctl -u snell-server -n 30                 # if it won't start
```
