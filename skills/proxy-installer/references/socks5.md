# s5-rust (SOCKS5)

Lightweight SOCKS5 proxy (username/password auth) installer. One-liner: `bash <(curl -Ls https://s.ee/socks5)`
Source: https://github.com/missuo/s5-rust · script `install.sh`

- **Linux + systemd only. Root required.** Needs `/etc/os-release` and `systemctl`.
- **Architectures:** amd64 · arm64.
- Only dependency is `curl` (to download the binary). No package installs.

## ⚠️ Not a GFW-bypass tool

Plain SOCKS5 offers **no encryption or obfuscation of its own** and **cannot penetrate the GFW**. **Do not recommend s5-rust for accessing the internet from mainland China through an overseas server** — it will be detected/blocked. Use OpenSnell, Shadowsocks-Rust, or Mieru for that. s5-rust is appropriate for: outbound-IP selection (`--send-through`), chaining behind another secure tunnel, LAN/trusted-network use, or region-hopping where censorship isn't the concern.

## ⚠️ No subcommands — interactive menu only

Unlike the other three installers, `s5-rust`'s script has **no CLI subcommands**. `main()` is an infinite menu loop (options 0–9: Install, Uninstall, Update, Start, Stop, Restart, Enable Autostart, Disable Autostart, Show Status, Exit). It also runs `exec < /dev/tty` when stdin is not a terminal.

Consequences for automation:
- `bash <(curl -Ls https://s.ee/socks5) install` → the `install` arg is **ignored**; you get the menu.
- Headless/piped with no controlling terminal → the `exec < /dev/tty` **fails and kills the script**.

So: use the one-liner **only** with a human at a real terminal (or a PTY via `script`/`expect`). For headless/agent installs, use the manual method below.

## Menu install prompts (interactive path)

Selecting `1) Install` prompts: reinstall confirm (if already installed) → `username` (blank = random 8) → `password` (blank = random 16) → `port` (blank = 1080) → `send-through CIDR` (blank = skip). It then downloads the binary, writes config + unit, and starts + enables the service.

## Paths

| Item | Path |
|------|------|
| Binary | `/usr/local/bin/s5-rust` |
| Config | `/etc/s5-rust/config` (`USERNAME`/`PASSWORD`/`PORT`/`SEND_THROUGH`) |
| systemd unit | `/etc/systemd/system/s5-rust.service` |
| Service name | `s5-rust` |

## Binary CLI flags

| Flag | Short | Meaning | Default |
|------|-------|---------|---------|
| `--username` | `-u` | Auth username (required) | — |
| `--password` | `-p` | Auth password (required) | — |
| `--port` | | Listen port | 1080 |
| `--bind` | | Bind address | 0.0.0.0 |
| `--send-through` | | Outbound source-IP CIDR | none |
| `--help` | `-h` | Help | |

`RUST_LOG=debug` enables debug logging.

**`--send-through`**: outbound connections use a source IP from the CIDR. A client can pin a specific source IP for one connection by appending `@<IP>` to its password (the IP must be inside the CIDR, else auth fails). So the configured password must not end in `@<IP-in-CIDR>`.

## Manual / headless install (recommended for agents)

Replicate what the menu does, without the TTY-locked loop:

```bash
set -e
ARCH=$(uname -m); case "$ARCH" in x86_64|amd64) A=amd64;; aarch64|arm64) A=arm64;; *) echo "unsupported"; exit 1;; esac
VER=$(curl -s https://api.github.com/repos/missuo/s5-rust/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
curl -L -o /usr/local/bin/s5-rust "https://github.com/missuo/s5-rust/releases/download/${VER}/s5-rust-linux-${A}"
chmod +x /usr/local/bin/s5-rust

USER=myuser; PASS=$(tr -dc 'a-zA-Z0-9' </dev/urandom | head -c 16); PORT=1080   # customize
mkdir -p /etc/s5-rust
cat > /etc/s5-rust/config <<EOF
USERNAME=${USER}
PASSWORD=${PASS}
PORT=${PORT}
SEND_THROUGH=
EOF
chmod 600 /etc/s5-rust/config

cat > /etc/systemd/system/s5-rust.service <<EOF
[Unit]
Description=s5-rust SOCKS5 Proxy Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/s5-rust -u ${USER} -p ${PASS} --port ${PORT}
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now s5-rust
```

Add `--send-through <CIDR>` to the `ExecStart` line to enable outbound-IP selection.

> Note: the password is embedded in `ExecStart`, so it is visible via `systemctl cat s5-rust` and `ps` to any local user.

## Test

```bash
curl --socks5 127.0.0.1:1080 --proxy-user <user>:<pass> http://httpbin.org/ip
curl --proxy socks5h://<user>:<pass>@<server-ip>:1080 http://httpbin.org/ip   # remote, DNS via proxy
```
