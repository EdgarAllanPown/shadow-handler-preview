> **Shadow Handler (README ONLY)**
>
> This is only the README for the **Shadow Handler** project. Its purpose is to present the work that has been completed on the project so far.
>
> Due to the nature of this project, repository access will only be granted to **certified security researchers who are also known and established within the security community**.
>
> Each request for repository access will be reviewed and handled **on a case-by-case basis**.
>
> For more information, please contact **konkar1337@gmail.com**.

 # Shadow Handler

Python C2 server with a Metasploit-style CLI: PSK-authenticated PowerShell agents, queued commands, Shadow post-ex modules, SQLite operation logging, and single-file HTML reports.

**Use only on systems you are authorised to test.**

## Table of Contents

- [Overview](#overview)
- [Installation](#installation)
- [Quick start (HTTP)](#quick-start-http)
- [Configuration](#configuration)
- [Transports](#transports)
- [Security model](#security-model)
- [PowerShell agent](#powershell-agent)
- [DNS setup](#dns-setup)
- [CLI reference](#cli-reference)
- [Session commands](#session-commands)
- [HTTP API](#http-api)
- [Shadow modules](#shadow-modules)
- [Logging and reports](#logging-and-reports)
- [Project layout](#project-layout)
- [Legal](#legal)

## Overview

1. Start a listener: `start <http|https|dns|smb>` (default port per [Configuration](#configuration)).
2. Build an implant: `generate <transport> <callback_host> <port>` using the **same port** the listener uses.
3. Run the printed **stager one-liner** on the target (all transports). It fetches `/stager` over that transport, runs the full agent (evasion, stage, check-in). Each successful stager fetch registers a new PSK.
4. `interact <session_id>` and issue commands (queued on the server, delivered on the next check-in).
5. Optional: `logging start` during an operation, then `report generate <session>`.

Agents embed a unique `agent_id` and PSK (`data/agent_psks.json`). Sessions expire after **300 seconds** without check-in (`AGENT_TIMEOUT` in config).

## Installation

```bash
pip install -r requirements.txt
python main.py
```

The repo includes `data/agent_psks.json` as an empty JSON object (`{}`). Each `/stager` download (and each `generate` that writes a direct-run `.ps1`) adds an `agent_id` and PSK there. On first use the handler also creates `loot/`, `data/logs/`, and `agents/output/` if missing. After real operations, treat `data/agent_psks.json` as confidential; reset it to `{}` before sharing the tree or use a private fork only.

**Dependencies** (`requirements.txt`): `flask`, `flask-cors`, `colorama`, `prompt-toolkit`, `pycryptodome`, `requests`, `dnslib`, `dnspython`.

**Migrate DLLs (only the `Migrate` shadow module):** The repository includes `payloads/migrate-x64.dll` and `payloads/migrate-x86.dll`. On **any** transport (`http`, `https`, `dns`, `smb`), the module downloads them via `Invoke-C2` to the same logical routes `GET /migrate-x64.dll` and `GET /migrate-x86.dll` (PSK in query): raw bytes on HTTP(S), `body_b64` over the DNS/SMB API tunnel. Before injection, `migrate.ps1` prefetches `GET /stager` via `Invoke-C2` and passes the staged agent into the DLL (`2|agent_id|psk|session_id|<b64>`), so migrated processes use the same staging flow as the operator one-liner. Rebuild with `dll_source/NativeBootstrap/compile_powerchell_dll.bat` (x64) and `compile_x86.bat` (x86); see `payloads/README.md` and `dll_source/README.md`. **`Process-Migration`** does not use these DLLs.

## Quick start (HTTP)

```text
handler > start http
handler > generate http 192.168.1.10 80 --retries 0
```

Deploy the printed stager one-liner (`powershell -nop -w h -ep b -e ...`). Optional: run `agents/output/agent_<timestamp>.ps1` directly (PSK shown at generate time).

```text
handler > sessions
handler > interact <session_id>
handler (<session_id>) > whoami
handler (<session_id>) > shadow run Screenshot --Interval=2 --Duration=10
```

Binding port **80** often requires an elevated shell. If you cannot use the default port, pick any free port for **both** `start http <port>` and `generate http <host> <port>` (see [Configuration](#configuration)).

## Configuration

All defaults below match `config/settings.py` and `Config.default_listener_port()`. Override listener ports with environment variables before starting the handler.

| Setting | Default | Env variable | Notes |
|---------|---------|--------------|--------|
| HTTP listener port | **80** | `HTTP_PORT` | Used when `start http` omits port |
| HTTPS listener port | **443** | `HTTPS_PORT` | Alias: `C2_PORT` |
| DNS listener port | **53** | `DNS_PORT` | UDP DNS |
| SMB-framed TCP port | **445** | `SMB_PORT` | Custom JSON framing, not Microsoft SMB/CIFS |
| DNS C2 zone | `shadow.c2.local` | `DNS_C2_ZONE` | Must match on `start dns` and `generate dns --zone` |
| Bind address | `0.0.0.0` | `C2_HOST` | |
| Agent check-in interval (new agents) | 5 s | (none) | `HTTP_CALLBACK_INTERVAL` |
| Jitter | 0.2 (20%) | (none) | `HTTP_JITTER` |
| Session timeout | 300 s | (none) | `AGENT_TIMEOUT` |
| Max HTTP request body | 500 MB | (none) | Flask `MAX_CONTENT_LENGTH` in `protocols/http_handler.py` |

**Paths** (under install root / `Config.BASE_DIR`):

| Path | Purpose |
|------|---------|
| `data/agent_psks.json` | Agent PSK registry (shipped as `{}`; filled on `generate` / `/stager`) |
| `data/logs/` | SQLite logging sessions (`session_*.db`) and default HTML reports |
| `agents/output/` | Generated `.ps1` implants |
| `loot/<agent_id>/` | Exfiltrated files and screenshots |
| `logs/handler.log` | Handler file log (`LOGS_DIR`) |

## Transports

Wire protocols for the PowerShell implant. Each transport implements the same logical API; HTTP(S) uses REST directly, DNS and SMB-framed TCP tunnel via `Invoke-C2`.

| Transport | Listener command | Default port |
|-----------|------------------|--------------|
| `http` | `start http [port]` | 80 |
| `https` | `start https [port] --cert <file> --key <file>` | 443 |
| `dns` | `start dns [port] [--zone <zone>]` | 53 |
| `smb` | `start smb [port]` | 445 |

```text
handler > stop http 80
handler > stop dns
```

`stop <transport>` with no port stops every listener of that transport.

**Notes**

- **All transports**: tunneled or direct `GET /stager` returns a new full PowerShell agent (new PSK per download). The generate one-liner is a small bootstrap (`stager_bootstrap.ps1` + transport layer) that calls `/stager` then runs the agent (evasion, stage, loop).
- **DNS**: Agents send UDP DNS to `callback_host`:`port` (custom port supported). Large API bodies use chunked `apc` / `apg`. See [DNS setup](#dns-setup).
- **SMB**: Port **445** conflicts with Windows file sharing on many hosts; stop the service, choose another port for **both** listener and `generate smb`, or set `SMB_PORT`.

## Security model

All transports share the same application logic: `C2Core` (stage / check-in) and `C2Services` (upload, screenshot, evasion, migration DLLs, API tunnel). DNS (`stg`, `chk`, `api`, `apc`/`apg`) and SMB-framed TCP call the same handlers as HTTP(S); only the wire encoding differs.

### Agent authentication (PSK)

Each implant gets a unique `agent_id` and PSK when `/stager` succeeds (staged deployment) or when you run a `generate` `.ps1` that was built with a pre-registered PSK. The handler stores keys in `data/agent_psks.json`.

- **Validation**: `psk_manager.validate_psk()`: unknown agent, wrong PSK, or **revoked** PSK leads to request rejected (`401` or equivalent error in tunneled responses).
- **Comparison**: constant-time `compare_digest` for the PSK string.
- **Revocation**: `psk revoke <agent_id>` / `psk revoke all` applies to every transport immediately.

Protected operations (require valid `agent_id` + PSK, and a valid `session_id` where noted):

| Operation | Extra checks |
|-----------|----------------|
| Stage (`/stage`, DNS `stg`) | (none) |
| Check-in (`/checkin`, DNS `chk`) | Active session |
| Upload / screenshot | Active session |
| Evasion script | (none) |
| Migration DLL download (`/migrate-*.dll`) | (none); **Migrate** module only; all transports via `Invoke-C2` |

Check-in and exfil paths also require a session created by a successful stage for that agent.

### Unauthenticated endpoints

| Endpoint | Transport | Behaviour |
|----------|-----------|-----------|
| `GET /health` | HTTP(S) only | No secret; liveness probe |
| `GET /stager` | All transports | No PSK; each response registers a **new** agent + PSK (staging vector). HTTP(S): plain GET. DNS/SMB: `Invoke-C2` API tunnel with `path=/stager`. |

### Transport security (wire vs application)

Application auth is the same; **confidentiality on the network** is not.

| Transport | PSK in protocol payloads | Encryption on the wire |
|-----------|--------------------------|-------------------------|
| `http` | Yes (JSON bodies) | No (cleartext HTTP) |
| `https` | Yes | Yes (TLS) |
| `dns` | Yes (QNAME / tunneled JSON) | No (cleartext DNS, UDP) |
| `smb` | Yes (length-prefixed JSON) | No (cleartext TCP) |

A passive observer on HTTP, DNS, or SMB can read PSKs and traffic unless you add external protections (VPN, isolated lab VLAN, etc.). Prefer **HTTPS** when you need TLS to the handler.

DNS chunked transfers (`apc` / `apg`) reuse the same authenticated API payloads split across queries; large responses are keyed by server-side `xfer_id` tied to the active transfer.

### Operational notes

- PSKs in generated scripts and logs are sensitive; restrict filesystem access to the handler host and operation databases/reports.
- This model is **engagement-scope access control**, not a substitute for legal authorisation, network segmentation, or hardened production deployment.

See also [HTTP API](#http-api) for the per-route auth summary.

## PowerShell agent

Single implant type: `agents/templates/powershell_agent.ps1` plus a transport layer in `agents/templates/transports/` (`http.ps1` for **both** `http` and `https`, `dns.ps1`, `smb.ps1`) implementing `Invoke-C2`. Shadow modules are `.ps1` under `modules/shadow/powershell/` and run in the agent process.

**Generate**

```text
generate <http|https|dns|smb> <callback_host> <port> [options]
```

| Option | Meaning |
|--------|---------|
| `--retries N` | Max staging reconnects (`0` = infinite, default) |
| `--obfuscate` | HTTP/HTTPS only: basic variable renaming |
| `--zone <name>` | DNS zone (default: `DNS_C2_ZONE`) |
| `--cert <file>` | HTTPS: server PEM used to embed the dev CA (`handler-ca.crt` in the same folder) |

Every `generate` writes `agents/output/agent_<timestamp>.ps1` (full agent with the PSK shown in the CLI), `agents/output/stager_<timestamp>.ps1` (same bootstrap as the one-liner), and prints the **stager one-liner** for the chosen transport: UTF-16LE base64 bootstrap (`agents/templates/stager_bootstrap.ps1` + the same transport layer as the full agent). The bootstrap loops on `Invoke-C2 GET /stager -RawText`, then `IEX` the full agent (safety checks, `GET /evasion`, `POST /stage`, check-in loop). Each successful `/stager` registers a **new** agent/PSK; the `.ps1` file is for direct run with the PSK from that generate command.

The printed one-liner is **only** the staged bootstrap (same as `stager_<timestamp>.ps1`), not the full agent. All transports use that path; the full implant is fetched at runtime via `Invoke-C2 GET /stager` (large DNS/SMB bodies are chunked inside `Invoke-C2`, not in the one-liner). If the base64 one-liner is too long for `cmd.exe`, run `agents/output/stager_<timestamp>.ps1` or start it from an open PowerShell window. Use `agent_<timestamp>.ps1` when you want a direct run with the PSK from that `generate` (no new PSK per `/stager`).

Optional AMSI/ETW bypass: agent-side safety checks, then `GET /evasion` (PSK-gated). Session list shows bypass status.

### HTTPS dev certificates

For lab TLS (not public PKI), create a private CA and server cert:

```text
python scripts/generate_dev_https_cert.py
```

This writes `data/certs/handler-ca.crt`, `handler.crt`, and `handler.key` (gitignored). `generate https … --cert data/certs/handler.crt` embeds the CA PEM (base64) in the agent. HTTPS `Invoke-C2` validates the server cert in memory against that CA only (no certificate store changes on disk).

```text
handler > start https 443 --cert data/certs/handler.crt --key data/certs/handler.key
handler > generate https 10.10.10.5 443 --cert data/certs/handler.crt
```

Use the same `handler.crt` for `start https` and `generate https --cert`.

## DNS setup

DNS agents do not use HTTP on the wire. They send **UDP DNS** queries to `DNS_SERVER`:`DNS_PORT` (custom port supported; not limited to 53). Names are under your **C2 zone**; the handler answers with `TXT` records. `generate dns <host> <port>` must match `start dns [port]`. The zone string must be identical on `start dns --zone …` and `generate dns … --zone …` (the listener normalizes the zone to lowercase).

### Lab zone (default port 53)

Private zone (default `shadow.c2.local`), no public DNS records required:

```text
handler > start dns --zone shadow.c2.local
handler > generate dns 10.10.10.5 53 --zone shadow.c2.local
```

Target runs the stager one-liner or `.ps1`. Example name shape: `<labels>.api.shadow.c2.local`.

Open **UDP 53** to the handler. Binding port 53 usually requires elevation on the handler host.

### Non-default DNS port

If you cannot bind to **53**, use the same alternate port on the listener, firewall, and `generate` line (not a framework default):

```text
handler > start dns 5353 --zone shadow.c2.local
handler > generate dns 10.10.10.5 5353 --zone shadow.c2.local
```

### Public subdomain (e.g. `c2.example.com`)

Delegate only the C2 subtree. Handler on public IP `203.0.113.10`:

```text
handler > start dns 53 --zone c2.example.com
handler > generate dns 203.0.113.10 53 --zone c2.example.com
```

At the DNS provider for `example.com`:

| Type | Name | Value |
|------|------|--------|
| A | `ns1.c2.example.com` | `203.0.113.10` |
| NS | `c2.example.com` | `ns1.c2.example.com` |

Add glue if the registrar requires it. No wildcard `*` is required; agents build dynamic labels under the zone.

Test from outside the LAN:

```bash
dig @8.8.8.8 probe.api.c2.example.com TXT +trace
```

Allow **UDP 53** to `203.0.113.10` (firewall / security groups).

### Enterprise conditional forwarder

If targets cannot use `-Server <handler-ip>`, forward **`c2.example.com`** (or your lab zone) internally to the handler. Agents still use `generate dns <ip> 53 --zone c2.example.com` (or your chosen port if not 53).

### Chunked DNS transfers

Single QNAME length is limited (~253 octets). Large check-ins and uploads are split automatically:

- **Inbound (`apc`)**: multiple queries reassembled on the handler.
- **Outbound (`apg`)**: large TXT responses split; agent fetches by `xfer_id`.

DNS is slow for bulk exfil; prefer HTTP or SMB-framed when possible.

## CLI reference

### Main context

| Command | Description |
|---------|-------------|
| `listeners` | List active listeners |
| `start <http\|https\|dns\|smb> [port] [--cert f --key f] [--zone z]` | Start listener (default port if omitted) |
| `stop <protocol> [port]` | Stop listener(s) |
| `sessions` | List sessions |
| `interact <session_id>` | Enter session context |
| `kill <session_id>` / `killall` | End session(s) |
| `generate <transport> <host> <port> [options]` | Build PowerShell agent |
| `exec_all <command>` | Run a session command on all active sessions |
| `loot [agent_id]` | List files under `loot/` |
| `shadow list\|info\|run\|load\|reload` | Shadow modules |
| `psk list [--all]\|info\|revoke\|stats` | PSK admin (`psk revoke all`) |
| `rename <agent_id> <name>` | Friendly name (persisted) |
| `logging start\|stop\|status\|list` | SQLite operation log |
| `report generate <session> [--output path]` | HTML report from a logging session |
| `info` | Handler statistics |
| `clear` / `help` / `exit` | Shell |

Tab completion: transports, sessions, shadow modules, `--retries`, `--obfuscate`, `--zone`, `--output`.

### Session context

`back` returns to main. Unknown bare tokens are treated as `shell` commands. See [Session commands](#session-commands).

## Session commands

| Command | Action |
|---------|--------|
| `shell <cmd>` | Run command |
| `cd <path>` / `ls [path]` / `pwd` | Filesystem |
| `download <remote>` | Target → handler via `/upload` → `loot/<agent_id>/` |
| `upload <local> <remote>` | Handler → target |
| `ps` | Process list |
| `whoami` / `hostname` / `getuid` / `getpid` / `sysinfo` | Recon |
| `sleep <seconds>` | Change agent check-in interval (same variable as staged `interval` / jitter loop) |
| `shadow run <module> [--param=value] [--timeout N]` | Post-ex module |
| `exit` | Kill agent |

## HTTP API

Used by HTTP(S) agents directly; DNS and SMB-framed agents use the same paths through `C2Services.handle_api` (`Invoke-C2` on the agent). Migration DLL downloads, `/upload`, `/screenshot`, `/evasion`, stage, and check-in all follow this model.

| Path | Method | Auth | Purpose |
|------|--------|------|---------|
| `/health` | GET | No | Health check |
| `/stager` | GET | No | New agent script (new PSK per download) |
| `/evasion` | GET | Query `agent_id`, `psk` | AMSI/ETW bypass script |
| `/stage` | POST | JSON `agent_id`, `psk` | Register session |
| `/checkin` | POST | JSON `agent_id`, `psk`, `session_id` | Heartbeat, queued commands, result upload |
| `/upload` | POST | JSON (or form on HTTP handler) | File exfil to `loot/` |
| `/screenshot` | POST | JSON `agent_id`, `psk`, `session_id`, … | Screenshot upload |
| `/migrate-x64.dll` / `/migrate-x86.dll` | GET | Query `agent_id`, `psk` | **`Migrate`** only; all transports (HTTP(S) raw body, DNS/SMB via tunneled API + `body_b64`) |

## Shadow modules

Directory: `modules/shadow/powershell/`. Each file includes a `.SHADOW_MODULE` metadata block (`Name`, `Description`, `Parameters`, etc.). Module names are **case-insensitive** in the CLI (`shadow run migrate` = `shadow run Migrate`).

Run: `shadow run <Name> [--timeout N] [--Param=value] …` (see `shadow info <Name>` for parameters).

| Name | File | What it does |
|------|------|----------------|
| `Domain-Creds-Hunter` | `domain-creds-hunter.ps1` | Phishes domain credentials until a target account type logs in |
| `Migrate` | `migrate.ps1` | `Invoke-C2` download of `migrate-*.dll` (any transport), then inject into a remote process |
| `Privesc-Auto` | `privesc-auto.ps1` | Automated privesc (StolenCreds, CMSTP UAC bypass, token duplication, or `Auto`) |
| `Process-Migration` | `process-migration.ps1` | Moves the agent into another process via COM, WMI, or `StartProcess` (no handler DLLs) |
| `Screenshot` | `screenshot.ps1` | Interval screenshots uploaded to the handler (`/screenshot`) |
| `Startup-Persistence` | `startup-persistence.ps1` | Startup persistence (WMI, scheduled task, or registry; ADS payload storage) |

The handler substitutes `{{CALLBACK_URL}}`, `{{AGENT_ID}}`, `{{PSK}}`, `{{SESSION_ID}}` before execution. When logging is active, parsed credential lines from module output (notably `Domain-Creds-Hunter`) are ingested into the `credentials` table on the CLI response path (`utils/shadow_output.py`).

## Logging and reports

```text
handler > logging start
handler > logging stop
handler > logging list
handler > report generate session_YYYYMMDD_HHMMSS
handler > report generate session_YYYYMMDD_HHMMSS --output C:\path\report.html
```

| Artifact | Location |
|----------|----------|
| Logging database | `data/logs/<session_name>.db` |
| Default HTML report | `data/logs/<session_name>_report.html` |

**Report sections:** compromised agents, command timeline (with output), screenshots (inline base64), exfiltrated files, harvested credentials, operational event timeline.

**SQLite tables:** `metadata`, `events`, `agents`, `sessions`, `commands`, `files`, `screenshots`, `credentials`, `exfiltration`. `/upload` during logging writes to `files` and `exfiltration`.

Reports are built by `modules/report_generator.py` (`report generate` in the CLI).

## Project layout

```text
main.py
config/settings.py
core/                 handler, session, PSK, c2_core, c2_services, dns_transfer_buffer
protocols/            http_handler, dns_handler, smb_handler
agents/               generator, templates/ (powershell_agent.ps1, transports/), output/
modules/              shadow_manager, logging_manager, report_generator.py, shadow/powershell/
ui/cli.py
utils/
evasion/amsi_etw_bypass.ps1
dll_source/           NativeBootstrap/ (C++ migration DLL sources and compile scripts)
payloads/             migrate-x64.dll, migrate-x86.dll (shipped; used by Migrate module)
data/                 agent_psks.json, logs/
loot/
logs/                 handler.log (file log; separate from data/logs/ SQLite)
```

## Legal

Authorized security testing and research only. See [DISCLAIMER.md](DISCLAIMER.md).
