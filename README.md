# bypass

One-line `curl`-to-`bash` installers for four self-hosted proxy servers, packaged as an installable **agent skill** plus an **[`llms.txt`](./llms.txt)** so any AI agent can install, configure, update, and manage them for you.

| Proxy | Install one-liner |
|-------|-------------------|
| OpenSnell / Snell | `bash <(curl -fsSL https://s.ee/opensnell)` |
| Shadowsocks-Rust | `bash <(curl -Ls https://s.ee/ssrust)` |
| Mieru | `bash <(curl -Ls https://s.ee/mieru)` |
| s5-rust (SOCKS5) | `bash <(curl -Ls https://s.ee/socks5)` |

All scripts run as **root** on **Linux + systemd**.

## 1. Installing the skill

The skill works with any agent supported by the [`skills`](https://github.com/vercel-labs/skills) CLI (Claude Code, Codex, Cursor, OpenCode, …). Install it with:

```bash
# Install into the current project
npx skills add missuo/bypass

# Or install globally (available in every project)
npx skills add missuo/bypass -g

# Non-interactive, e.g. into Claude Code only
npx skills add missuo/bypass --skill proxy-installer -a claude-code -y
```

This installs a single skill named **`proxy-installer`** (it bundles a reference file per proxy).

## 2. Using the skill

Once installed, just ask your agent in plain language — the skill activates automatically. For example:

- "Install a Shadowsocks server on this VPS and give me the share link."
- "Set up OpenSnell and print the Surge proxy line."
- "Deploy a Mieru proxy with two users over TCP."
- "Show me the connection info for the proxy running on this box."
- "The install script failed because `unzip` is missing — fix it and retry."

The agent will pick the right installer, drive it (interactively or headlessly), handle missing dependencies, and report back the credentials and client share links. It also knows the important gotchas — e.g. SOCKS5 has no subcommands and cannot bypass the GFW, and Mieru's daemon needs a two-step start.

## 3. Without installing the skill

If you'd rather not install anything, point your agent at the raw **`llms.txt`** — it contains the full operating knowledge on its own:

```
https://raw.githubusercontent.com/missuo/bypass/main/llms.txt
```

Paste that URL (or its contents) into any agent and it can operate all four installers directly. Per-tool reference files are linked from inside `llms.txt`.

## Reference

**Upstream software (third-party projects):**

- Shadowsocks-Rust — https://github.com/shadowsocks/shadowsocks-rust
- Mieru — https://github.com/enfein/mieru

**Author's own projects:**

- OpenSnell — https://github.com/missuo/opensnell
- s5-rust (SOCKS5) — https://github.com/missuo/s5-rust
