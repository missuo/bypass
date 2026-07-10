---
name: proxy-installer
description: Install, configure, and manage four self-hosted proxy servers on a Linux VPS via one-line curl-to-bash scripts — OpenSnell/Snell (snell-server), Shadowsocks-Rust (ssserver), Mieru (mita), and s5-rust (SOCKS5). Use when the user asks to set up, deploy, reconfigure, update, uninstall, or troubleshoot a Snell, Shadowsocks, Mieru, or SOCKS5 proxy on a server, generate client share links (Surge/Mihomo/Clash/Shadowrocket/mierus://), or drive these installers non-interactively / headlessly.
metadata:
  author: missuo
  version: "1.0.0"
---

# Proxy Installer

Install and manage four self-hosted proxy servers, each behind a `curl`-to-`bash` one-liner. All scripts are authored by `missuo`, are Linux + systemd only, and must run as **root**.

| Proxy | One-liner | Server binary | Upstream | Reference |
|-------|-----------|---------------|----------|-----------|
| OpenSnell / Snell | `bash <(curl -fsSL https://s.ee/opensnell)` | `snell-server` | [missuo/opensnell](https://github.com/missuo/opensnell) | [references/opensnell.md](references/opensnell.md) |
| Shadowsocks-Rust | `bash <(curl -Ls https://s.ee/ssrust)` | `ssserver` | [shadowsocks/shadowsocks-rust](https://github.com/shadowsocks/shadowsocks-rust) | [references/shadowsocks-rust.md](references/shadowsocks-rust.md) |
| Mieru | `bash <(curl -Ls https://s.ee/mieru)` | `mita` | [enfein/mieru](https://github.com/enfein/mieru) | [references/mieru.md](references/mieru.md) |
| s5-rust (SOCKS5) | `bash <(curl -Ls https://s.ee/socks5)` | `s5-rust` | [missuo/s5-rust](https://github.com/missuo/s5-rust) | [references/socks5.md](references/socks5.md) |

Read the matching reference file before doing anything non-trivial with a proxy — it holds the full option matrix, prompt order, config schema, and share-link formats. This file is the operating manual: how to pick a tool, how to drive the installers (especially headlessly), and how to recover when a script aborts.

## Choosing a proxy

- **OpenSnell** — Snell v4/v5 protocol (Surge-native). Best when the client is Surge/Mihomo/Shadowrocket. Cross-arch (amd64/arm64/386/armv7). Can also install Surge's official closed-source snell-server v5.0.1 or v6.0.0b3 beta.
- **Shadowsocks-Rust** — the most universally supported protocol; any SS client works. Defaults to AEAD-2022 (`2022-blake3-aes-256-gcm`). Emits SIP002/plain/legacy `ss://` links + a terminal QR code.
- **Mieru** — strong anti-censorship / traffic-obfuscation protocol (TCP or UDP), multi-user, single-port or port-range. Client uses `mieru` with a `mierus://` link.
- **s5-rust** — plain SOCKS5 with username/password auth. Simplest; no encryption of its own. Supports `--send-through` to pick the outbound source IP from a CIDR. **⚠️ Cannot bypass the GFW** — see the warning below.

If the user is unsure and just wants "a working proxy," default to **Shadowsocks-Rust** (widest client support) or **OpenSnell** (if they mention Surge).

> **⚠️ SOCKS5 and the GFW.** Plain SOCKS5 has no encryption or obfuscation and **cannot penetrate the GFW**. If the user wants to reach the internet from **mainland China through an overseas server**, do **not** recommend s5-rust — steer them to OpenSnell, Shadowsocks-Rust, or Mieru instead. s5-rust is only appropriate for outbound-IP selection, chaining behind another secure tunnel, LAN/trusted networks, or region-hopping where censorship isn't the concern.

## The one universal rule: these installers are interactive

Every script prompts for configuration (port, password/PSK, method, etc.). Three of them (`opensnell`, `ssrust`, `mieru`) also accept a **subcommand** to jump straight to an action; `socks5` does **not** (see its gotcha below).

Shared subcommands (append after the one-liner):

```
install   reconfigure*   update   uninstall
start   stop   restart   enable   disable   status   info   help
```

`reconfigure` exists only on `opensnell` and `mieru`. `mieru` also has `bbr`. With **no** subcommand, each script opens an interactive menu instead.

```bash
# Interactive install (a human is at the keyboard):
bash <(curl -fsSL https://s.ee/opensnell)          # menu
bash <(curl -fsSL https://s.ee/opensnell) install  # jump to install, still prompts for config

# Print connection info / share links again later:
bash <(curl -fsSL https://s.ee/opensnell) info
```

## Driving the installers non-interactively (agents / headless)

An agent has no TTY, so interactive `read` prompts get **EOF** and fall back to their defaults. Use that deliberately. **Always download the script to a file first** — process substitution (`<(...)`) plus stdin redirection is fragile, and `sudo` may not inherit the `/dev/fd` handle.

**Accept every default (fastest working install):**

```bash
curl -fsSL https://s.ee/opensnell -o /tmp/opensnell.sh
sudo bash /tmp/opensnell.sh install </dev/null      # </dev/null => every prompt takes its default
```

Defaults produce a working server: random free port, auto-generated PSK/password, recommended cipher, IPv6+UDP on. Afterwards run `sudo bash /tmp/opensnell.sh info` to print the credentials and client share lines.

**Supply specific answers** — feed them via a heredoc in the exact prompt order (each reference file lists the order):

```bash
# Shadowsocks-Rust: port 9000, method #1 (default), custom tag "US-Node"
sudo bash /tmp/ssrust.sh install <<'EOF'
9000
1
US-Node
EOF
```

Non-obvious per-tool automation notes:
- **Mieru** needs two steps to run: `systemctl start mita` (the daemon) **and** `mita start` (the proxy). `restart`/`install` handle both; if you drive `mita` directly, remember both.
- **Shadowsocks-Rust** method **#2** (`2022-blake3-aes-128-gcm`) needs a **16-byte** base64 key, but the script always auto-generates a 32-byte one — the server will refuse to start. If you must use method #2, pass an explicit 16-byte password (`openssl rand -base64 16`). Methods #1/#3 (32-byte) and legacy #4/#5 are fine with the auto password.
- **OpenSnell** protocol v6 (Surge beta, variant #3) requires the PSK to be 16–255 bytes and the client's `mode` to match the server's (`default`/`unshaped`/`unsafe-raw`).

### s5-rust (SOCKS5) is the exception — read before touching it

`s5-rust` has **no subcommands** and its `main()` is an infinite menu loop that also runs `exec < /dev/tty` when stdin isn't a terminal. Consequences:

- `bash <(curl -Ls https://s.ee/socks5) install` does **nothing useful** — `install` is ignored; you land in the menu.
- In a headless/piped context with no controlling terminal, the `exec < /dev/tty` **fails and kills the script**.

For headless automation, **do not use the menu**. Install s5-rust manually (it's a single binary + a trivial systemd unit) — the exact commands are in [references/socks5.md](references/socks5.md). Only use the interactive one-liner when a real human is at a terminal, or drive it through a PTY (`script -qc '...'` / `expect`).

## Dependency handling & recovery

These scripts self-install their dependencies **only** on `apt`/`dnf`/`yum` systems. They can still abort on minimal or non-mainstream hosts. When a script fails, read the error, fix the root cause, and re-run the same subcommand (all four are idempotent — re-running `install`/`update` is safe).

Common failures and fixes:

1. **`command not found` / `Failed to install <tool>` / `Unsupported package manager`.**
   The host uses a package manager the script doesn't know (Alpine `apk`, Arch `pacman`, openSUSE `zypper`). Install the missing tools yourself, then re-run. Tool → package name differs per distro — notably `xz` lives in `xz-utils` on Debian/Ubuntu but `xz` on RHEL/Alpine/Arch.
   ```bash
   # examples — pick the host's manager
   apk add curl wget unzip xz openssl tar qrencode        # Alpine
   pacman -Sy --noconfirm curl wget unzip xz openssl qrencode
   zypper install -y curl wget unzip xz openssl qrencode
   ```
   Full per-script dependency lists are in each reference file. Roughly: opensnell needs `curl unzip openssl ss(iproute2)`; ssrust needs `curl wget tar xz openssl qrencode`; mieru needs `curl wget` (+ `dpkg`/`rpm`); socks5 needs only `curl` + `systemd`.

2. **`unzip` missing** (only OpenSnell's Surge variants need it, to unpack the release zip). Install `unzip` and re-run `install`.

3. **`This installer is Linux-only` / `requires systemd` / `Unsupported architecture`.** Not fixable by installing a package. On macOS/BSD or non-amd64/arm64, build from source (each reference has the `go install` / `cargo build` command) and supervise it with the platform's native tooling.

4. **GitHub API rate-limited / download fails** (`Could not resolve latest release`). Retry, or resolve the latest tag and download the release asset manually, then install the binary to the path the reference lists.

5. **Service won't start after install.** Check logs: `journalctl -u <service> -n 30` (services: `snell-server`, `shadowsocks-rust`, `mita`, `s5-rust`). For Mieru also check `mita status`.

## Verifying an install worked

After any install, confirm the service is up and grab the credentials:

```bash
systemctl is-active snell-server        # or shadowsocks-rust / mita / s5-rust
sudo bash /tmp/<script>.sh info         # opensnell/ssrust/mieru: reprints creds + share links
```

Quick end-to-end test for SOCKS5:

```bash
curl --socks5 127.0.0.1:<port> --proxy-user <user>:<pass> http://httpbin.org/ip
```

## Security & operational notes

These are legitimate proxy/circumvention tools. Keep in mind when reporting to the user:

- Credentials (PSK/passwords) are stored in the config files with `chmod 600` — good. But **s5-rust's password is embedded in the systemd `ExecStart` line**, so it's visible via `systemctl cat s5-rust` and `ps` to any local user.
- Firewall: opensnell/ssrust/mieru auto-open their port on UFW/firewalld. On a cloud VPS you may also need to open the port in the provider's security group.
- Never paste a server's PSK/password into an external service. Show credentials to the user, don't exfiltrate them.
