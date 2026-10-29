# Mieru (`mita`)

Mieru server (`mita`) installer. One-liner: `bash <(curl -Ls https://s.ee/mieru)`
Script source: https://github.com/missuo/snell-server-docker (`install-mieru.sh`)
Upstream software: https://github.com/enfein/mieru (installed as a `.deb`/`.rpm` package)

- **Linux + systemd only. Root required.**
- **OS families:** Debian/Ubuntu (`dpkg`/`apt`) **or** RHEL/Fedora/CentOS (`rpm` + `dnf`/`yum`). Other families abort — install the package manually from the upstream releases.
- **Architectures:** amd64/x86_64 · arm64/aarch64.
- Mieru is a strong anti-censorship / traffic-obfuscation protocol; good choice for hostile networks.

## Paths

| Item | Path |
|------|------|
| Config | `/etc/mita/server_config.json` |
| Install metadata | `/etc/mita/.install_meta` |
| Service name | `mita` (systemd unit shipped by the package) |
| Binary | `mita` (on `PATH`, installed by the package) |

## Subcommands

`install` · `reconfigure` · `update` · `uninstall` · `start` · `stop` · `restart` · `enable` · `disable` · `status` · `info` · `bbr` (alias `enable-bbr`) · `help`. No argument → interactive menu (options 0–12).

## Dependencies

`check_requirements` installs `curl`, `wget`, `openssl` via apt/dnf/yum. The package install itself uses `dpkg`/`apt` or `dnf`/`yum`/`rpm`.

## ⚠️ Two-step service model (important)

The `.deb`/`.rpm` ships a systemd unit named `mita` that runs the **daemon**; the **proxy** is then started via the `mita` CLI. So a running proxy needs **both**:

```bash
systemctl start mita     # start the daemon
mita start               # tell the daemon to start proxying
```

`install`, `restart`, and `reconfigure` already do both and warn if `mita status` does not reach `RUNNING`. If you drive `mita` directly, don't forget the CLI `start`/`stop`.

## Config prompts (in order) — for scripting a heredoc

1. Transport protocol `Enter your choice (1-2, default: 1):` → 1 = `TCP` (recommended), 2 = `UDP`
2. Port mode `Enter your choice (1-2, default: 1):` → 1 = single port, 2 = port range
3a. (single) `Enter server port (default: <random 30000–50000>):` — must be 1025–65535
3b. (range) `Enter port range (e.g. 2012-2022, default: 2012-2022):`
4. `Number of users (default: 1):`
5. Per user: `Username (default: mieru<random4>):` then `Password (leave empty for random):` (random = 24 chars)
6. `Server address (default: auto-detect public IPv4):` — the address clients connect to, used only for the share link. **Set it behind NAT / relay / IPLC entries**, where the detected egress IP is wrong. Saved and reused by `info`; `MIERU_SERVER_ADDR=<addr>` overrides it for both the prompt default and `info`.
7. `Listen IP address (leave empty to listen on all interfaces):` → `listenIPAddress` (mita v3.37.0+)
8. Low entropy mode `Enter your choice (1-5, default: 1):` → 1 = off, 2/3/4/5 = `LOW_ENTROPY_MODE_32/40/48/56` (v3.35.0+). Lowers ciphertext entropy against DPI at the cost of traffic/CPU (32 ≈ +100%, 56 ≈ +14%). A random `LOW_ENTROPY_MASK_ROTATE_RIGHT_1..15` rotation is picked.
9. `Customize padding lengths? (y/N):` → if `y`: `Max middle padding bytes (0-255, 0 disables):` then `Max end padding bytes (0-255, 0 disables):` (v3.34.0+; blank keeps mieru's internal value)
10. `TCP idle timeout, e.g. 15m or 1h (min 1m; leave empty for none):` → `advancedSettings.tcpIdleTimeout` (v3.39.0+)
11. `Require user hint, rejecting clients older than v3.31.0? (y/N):` → `advancedSettings.userHintIsMandatory`

`</dev/null` gives: TCP, single random port, one user with random name/password, auto-detected address, all optional features off.

Traffic patterns (low entropy, padding) are server-side only here: client and server pick their own patterns independently and any receiver accepts all valid modes, so turning them on does not break existing clients.

## Resulting config file

```json
{
    "portBindings": [
        { "port": 40000, "protocol": "TCP" }
    ],
    "users": [
        { "name": "mieruXXXX", "password": "<random>" }
    ],
    "loggingLevel": "INFO",
    "mtu": 1400
}
```

Optional fields appear only when chosen: `listenIPAddress`, `advancedSettings` (`tcpIdleTimeout`, `userHintIsMandatory`), `trafficPattern` (`lowEntropy`, `padding`).

Written with `mita replace config /etc/mita/server_config.json` (v3.38.0+), which replaces the whole saved config — users dropped from the file are really removed. Older mita falls back to `mita apply config`, which **merges users by name** and keeps removed ones (the script warns). Changing `listenIPAddress` needs `mita stop` + `mita start`; `reconfigure` restarts the daemon, which covers it.

## BBR

`bbr` subcommand enables BBR congestion control (`net.core.default_qdisc=fq`, `net.ipv4.tcp_congestion_control=bbr`). Skips if BBR is already active, leaving existing setups (e.g. BBR+CAKE) untouched.

## Firewall

Opens the port/range on UFW or firewalld (protocol matches TCP/UDP; a range like `2012-2022` becomes `2012:2022` on UFW).

## Output / client share format (`info` reprints these)

- Server address (saved answer / `MIERU_SERVER_ADDR` / detected IPv4), port, protocol, any enabled optional features, and every user's name/password.
- **`mierus://` share link** per user:
  `mierus://<user>:<pass>@<addr>?profile=default&port=<port>&protocol=<TCP|UDP>&mtu=<mtu>`
- Client side: `mieru import config <URL>`, then `mieru start`.

## Common operations

```bash
curl -Ls https://s.ee/mieru -o /tmp/mieru.sh
sudo bash /tmp/mieru.sh install </dev/null    # TCP, random port, 1 random user
sudo bash /tmp/mieru.sh info                  # print users + mierus:// links
MIERU_SERVER_ADDR=1.2.3.4 sudo -E bash /tmp/mieru.sh info   # links with a NAT/relay entry address
sudo bash /tmp/mieru.sh reconfigure           # rebuild config + re-apply
sudo bash /tmp/mieru.sh bbr                    # enable BBR
sudo bash /tmp/mieru.sh update
sudo bash /tmp/mieru.sh uninstall
systemctl is-active mita && mita status
```
