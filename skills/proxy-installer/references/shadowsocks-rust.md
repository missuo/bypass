# Shadowsocks-Rust (`ssserver`)

Shadowsocks-Rust server installer. One-liner: `bash <(curl -Ls https://s.ee/ssrust)`
Script source: https://github.com/missuo/snell-server-docker (`install-ssrust.sh`)
Upstream software: https://github.com/shadowsocks/shadowsocks-rust (musl static binary)

- **Linux + systemd only. Root required.**
- **Architectures:** x86_64 / aarch64.
- Downloads `shadowsocks-<ver>.<arch>-unknown-linux-musl.tar.xz` from the upstream releases.

## Paths

| Item | Path |
|------|------|
| Binary | `/usr/local/bin/ssserver` |
| Config | `/etc/shadowsocks-rust/config.json` |
| Node tag | `/etc/shadowsocks-rust/.node_tag` |
| systemd unit | `/etc/systemd/system/shadowsocks-rust.service` |
| Service name | `shadowsocks-rust` |

## Subcommands

`install` · `update` · `uninstall` · `start` · `stop` · `restart` · `enable` · `disable` · `status` · `info` · `help`. No argument → interactive menu (options 0–10). **No `reconfigure`** — to change settings, edit `config.json` and `restart`, or re-run `install`.

## Dependencies

`check_requirements` installs, via apt/dnf/yum (adds `epel-release` on dnf/yum): `curl`, `wget`, `tar`, `xz` (package `xz-utils` on Debian/Ubuntu, `xz` on RHEL), `openssl`, `qrencode`. Install manually on other package managers.

## Config prompts (in order) — for scripting a heredoc

1. `Enter server port (default: 8388):`
2. Encryption method `Enter your choice (1-5, default: 1):`
   1. `2022-blake3-aes-256-gcm` (recommended, AEAD-2022)
   2. `2022-blake3-aes-128-gcm`
   3. `2022-blake3-chacha20-poly1305`
   4. `chacha20-ietf-poly1305` (legacy)
   5. `aes-256-gcm` (legacy)
3. `Enter node name/tag (leave empty for random):` → blank = `SS-<random6>`

The password is always auto-generated (`openssl rand -base64 32`). There is no password prompt.

### ⚠️ Method #2 caveat (real gotcha)
`2022-blake3-aes-128-gcm` needs a **16-byte** base64 key, but the script always generates a **32-byte** key — with method #2 the server will refuse to start. If you need method #2, after install replace `"password"` in `config.json` with `openssl rand -base64 16` and `restart`. Methods #1 and #3 (32-byte) and legacy #4/#5 (arbitrary password) work with the auto-generated key.

## Resulting config file

```json
{
    "server": "0.0.0.0",
    "server_port": 8388,
    "password": "<base64>",
    "method": "2022-blake3-aes-256-gcm",
    "mode": "tcp_and_udp",
    "fast_open": true
}
```

Install also enables system TCP Fast Open (`net.ipv4.tcp_fastopen = 3` in `/etc/sysctl.conf`).

## Firewall

Opens the port on **UFW only** (no firewalld support). Open it in the cloud provider's security group as well.

## Output / client share formats (`info` reprints these)

- Server address / port / password / method / tag.
- **SIP002** (recommended): `ss://<base64url(method:password)>@<ip>:<port>#<tag>`
- **Plain**: `ss://<method>:<urlenc(password)>@<ip>:<port>#<tag>`
- **Legacy base64**: `ss://<base64(method:password@ip:port)>#<tag>`
- A terminal **QR code** of the SIP002 URI (via `qrencode -t UTF8`).

## Common operations

```bash
curl -Ls https://s.ee/ssrust -o /tmp/ssrust.sh
sudo bash /tmp/ssrust.sh install </dev/null   # default: port 8388, method #1, random tag
# custom (port 9000, method #1, tag US-Node):
sudo bash /tmp/ssrust.sh install <<'EOF'
9000
1
US-Node
EOF
sudo bash /tmp/ssrust.sh info                 # print creds + ss:// links + QR
sudo bash /tmp/ssrust.sh update               # upgrade to latest upstream release
sudo bash /tmp/ssrust.sh uninstall
systemctl is-active shadowsocks-rust
journalctl -u shadowsocks-rust -n 30
```
