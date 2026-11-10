# pcp-terminal

Lightweight C++ client for **pcp-chat (cli edition)**.

**Why it exists:** one of the main goals is to use your chat **from any remote
machine while your account database stays at home**. The full client owns all
data; a relay started next to it holds sessions **only in RAM** (`:memory:`).
So the worst case is your account data existing *temporarily, in the memory of
the remote client* — never persisted on the machine you are sitting at.

It is a **smart `curl` wrapper**: it never embeds the Matrix stack. Instead it
talks over plain HTTP to a `pcp-cli serve --ttys` relay and renders the
server's ASCII UI on your terminal. That keeps the local binary small and fast
to build — perfect when you want the remote ASCII experience without shipping
the heavy client.

## What it does

- **Detects your terminal size** (`TIOCGWINSZ` → `COLUMNS`/`LINES` → 80×24).
- Sends it as `term:{cols,rows}` in the request.
- Renders the server's ASCII UI frame and prints it.
- Offers the full flow without a big binary: `register`, `session`, `input`,
  `sync`, plus profile containers (`profile …`), relay proxy control
  (`proxy on/off/status`) and `logout` — all over the same HTTP API.

## Build

```bash
cmake -S . -B build
cmake --build build
```

Interactive mode lives in the **`dumb` branch** (`progterm term <session>`),
not here — this build is purely the command set.

## Profiles (account containers) and per-profile proxy

`pcp-terminal` uses **profiles** — containers that hold an optional
account (session) plus connection settings such as a per-profile `proxy`. A
profile may exist with **no account** (just proxy/host config), and **at least
one profile is always enabled**:

```
~/.config/pcp-terminal/profiles/<name>   # key=value: enabled/proxy/host/session
~/.config/pcp-terminal/current          # active profile name (must be enabled)
```

This is the **only** state the client keeps on disk — no message database, no
Matrix data. The `proxy` is per-profile and forwarded to the server, which
applies it to that profile's homeserver traffic (overriding the server-wide
default). Use `off` for a direct connection.

```bash
# Profiles (containers for accounts + settings):
pcp-terminal profile create work --proxy socks5://127.0.0.1:9050
pcp-terminal profile create personal
pcp-terminal profile enable work | disable work   # >=1 stays enabled
pcp-terminal profile current personal              # active profile
pcp-terminal profile list

# Accounts live inside profiles:
pcp-terminal register --profile personal --homeserver https://a.org \
    --username alice --password secret --proxy off
pcp-terminal session  --profile bob --homeserver https://b.org \
    --user @bob:b.org --token TOK --device DEV --proxy socks5://127.0.0.1:9050

pcp-terminal render --static          # uses the active profile
pcp-terminal render --profile bob     # or pick one explicitly
pcp-terminal profile current personal  # switch active profile
pcp-terminal logout [--profile bob]   # forget an account (keep profile)

# Drive the relay's proxy (the full client's `proxy on/off`) over HTTP:
pcp-terminal proxy on tor
pcp-terminal proxy off
pcp-terminal proxy status             # local profile + relay status
```

`render` / `input` / `sync` default to the active profile unless `--profile`
is given. The active profile's host also acts as a `PROGTERM_HOST` fallback.

## Relay authentication

When the relay runs with `serve --ttys --token <t>`, every command accepts
`--relay-token <t>` (or the `PROGTERM_TOKEN` environment variable) and sends
it as an `Authorization: Bearer` header:

```bash
pcp-terminal proxy status --relay-token secret
PROGTERM_TOKEN=secret pcp-terminal render --static
```

Without a token (relay started without `--token`) no header is sent and no
auth is required.

The server side (`pcp-cli serve --ttys`) still holds all real state; by
default its database is in-memory (`:memory:`), so a session lives in the
server's RAM and is lost when that server restarts.

## Auto-connect (port scan)

When no host is given explicitly (`--host` / `PROGTERM_HOST` / cached account
host are all absent), the client scans `127.0.0.1` ports near a base port and
connects to the first `pcp-cli serve --ttys` relay it finds. The scan
expands outward from the base (base, +1, −1, +2, …) up to `--scan-range`
(default 10), so a relay running on `29325 ± N` is discovered without any
configuration:

```bash
pcp-terminal register \
    --homeserver https://example.org --username alice --password secret
# ^ no --host: auto-scans 127.0.0.1:29315..29335, connects to the relay

pcp-terminal render --static   # also auto-discovers when needed
```

Tuning / disabling:

```bash
pcp-terminal register ... --scan-base 29325 --scan-range 20
pcp-terminal render --no-scan   # never scan; require an explicit host
```

The scan only probes localhost, so it is fast and safe; a found relay is
remembered per-profile after the first successful `register`/`session`.

## Usage

```bash
# Point at a running relay (or set PROGTERM_HOST):
export PROGTERM_HOST=http://127.0.0.1:29325

# 1. Register a fresh account (prints a session id + credentials):
pcp-terminal register \
    --homeserver https://example.org --username newuser --password secret

# 2. Or reuse an existing account:
pcp-terminal session \
    --homeserver https://example.org --user @me:example.org \
    --token TOKEN --device DEV

# 3. Render the ASCII UI (auto-detects your terminal size):
pcp-terminal render --static   # session comes from the active profile

# 4. Interactive remote terminal: dumb branch, `progterm term <session>`.
```

## Server side

You still need a `pcp-cli` relay somewhere (a small box, a container, a
remote host) running:

```bash
pcp-cli serve --ttys --port 29325
```

`pcp-terminal` is purely the client.

## License

See repository settings.
