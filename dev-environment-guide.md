# VPS Developer Environment — Complete Setup Guide

This guide consolidates everything from the three setup sessions on **26 Nov 2026**:
installing GitHub CLI and connecting the GitHub account, building a standard developer
home layout, installing/updating **Hermes Agent**, and setting up **Free Claude Code**
with a single-source API key.

The machine is an Ubuntu 26.04 VPS, user `ubuntu`, with passwordless `sudo`, home
directory `/home/ubuntu`.

---

## 0. What is installed at a glance

| Component | Version / location | Purpose |
|---|---|---|
| GitHub CLI (`gh`) | **2.106.0**, `/usr/bin/gh` | Repos, auth, PRs, API |
| Git | system package | Version control |
| Hermes Agent | **v0.23.4** (stable), `~/.local/bin/hermes` | General-purpose AI agent |
| Free Claude Code (FCC) | **6.19.56**, `~/.local/bin/fcc-server` | Local Anthropic-compatible proxy |
| Claude Code CLI | **2.1.332**, `~/.local/bin/claude` | Coding CLI, pointed at FCC |
| OpenRouter key | `~/.secrets/openrouter.env` (mode `600`) | Single source for both agents |
| FCC service | `fcc-server.service` (systemd --user) | Runs proxy on `127.0.0.1:8082` |
| Main repo | `~/testingvps` → `github.com/laurentHYZ/testingvps` | Bound to the private test repo |

**Quick health check:**

```bash
gh --version
hermes --version
fcc-server --version
claude --version
systemctl --user is-active fcc-server
curl -s http://127.0.0.1:8082/health
```

Expected: `active`, and a JSON health response from port `8082`.

---

## 1. GitHub CLI

### 1.1 Install from GitHub's official apt repository

The distro package was too old (2.46, early 2024), so GitHub's own repository was used
instead. This also means normal `apt upgrade` keeps `gh` current.

```bash
set -e
sudo mkdir -p -m 755 /etc/apt/keyrings

wget -qO- https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null
sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
  | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null

sudo apt-get update
sudo apt-get install -y gh
```

Verify:

```bash
gh --version      # gh version 2.106.0
```

### 1.2 Authenticate

The device/web flow is the safest option on a headless VPS: it never puts a token in
the chat or shell history.

```bash
gh auth login --hostname github.com --git-protocol https --web
```

On a machine with no browser, start it in the background and read the one-time code:

```bash
mkdir -p /tmp/opencode
nohup gh auth login --hostname github.com --git-protocol https --web \
  > /tmp/opencode/gh-login.log 2>&1 < /dev/null &

sleep 5
cat /tmp/opencode/gh-login.log
```

Then, on any device with a browser:

1. Open **https://github.com/login/device**
2. Enter the one-time code printed in the log
3. Approve access for GitHub CLI

Confirm:

```bash
gh auth status
```

Result in this setup: logged in as **`laurentHYZ`**, git protocol **HTTPS**, scopes
`repo`, `read:org`, `gist` — enough for private repos, org reads, and gists.

> The token is stored by `gh` in `~/.config/gh/hosts.yml`. That is `gh`'s normal
> credential store; treat that file as a secret.

### 1.3 Create the private repo and bind a local folder

```bash
gh repo create testingvps --private
gh repo clone laurentHYZ/testingvps testingvps
```

Verify the remote and branch:

```bash
git -C ~/testingvps remote -v
git -C ~/testingvps status -sb
```

- Folder: `/home/ubuntu/testingvps`
- Remote `origin`: `https://github.com/laurentHYZ/testingvps.git`
- Branch: `main` (empty at creation, so `origin/main` does not exist until the first push)

Useful repo commands:

```bash
gh repo view laurentHYZ/testingvps --json name,visibility,url,isEmpty
gh repo list --limit 10
```

---

## 2. Standard developer home layout

The home directory was populated with the usual XDG folders plus developer-oriented
buckets:

```bash
cd ~
mkdir -p \
  Desktop Documents/{Personal,Work} Downloads Music \
  Pictures/Screenshots Public Templates Videos/{Recordings,Captures} \
  Projects/{active,archive,experiments} Scripts Notes Archive Sandbox bin
```

Resulting tree (hidden config folders omitted):

```
Archive/
Desktop/
Documents/
├── Personal/
└── Work/
Downloads/
Music/
Notes/
Pictures/
└── Screenshots/
Projects/
├── active/
├── archive/
└── experiments/
Public/
Sandbox/
Scripts/
Templates/
Videos/
├── Captures/
└── Recordings/
bin/
testingvps/        # git repo, bound to GitHub
```

### 2.1 XDG folder mapping

`~/.config/user-dirs.dirs` was written so GUI file managers point at the same folders:

```ini
XDG_DESKTOP_DIR="$HOME/Desktop"
XDG_DOWNLOAD_DIR="$HOME/Downloads"
XDG_TEMPLATES_DIR="$HOME/Templates"
XDG_PUBLICSHARE_DIR="$HOME/Public"
XDG_DOCUMENTS_DIR="$HOME/Documents"
XDG_MUSIC_DIR="$HOME/Music"
XDG_PICTURES_DIR="$HOME/Pictures"
XDG_VIDEOS_DIR="$HOME/Videos"
```

### 2.2 `~/bin` on PATH

`~/bin` is already added to `PATH` by `~/.profile` for login shells. Drop scripts
there and restart the shell once. `~/Scripts` is for grouped utilities; `~/Sandbox`
for throwaway tests.

---

## 3. Hermes Agent (Nous Research)

### 3.1 What was already there

Hermes was already installed; it did **not** need a fresh install.

| Item | Value |
|---|---|
| Version | **v0.23.4** (2026.9.24), stable channel |
| Install method | Git clone at `~/.hermes/hermes-agent` |
| Upstream | `https://github.com/NousResearch/hermes-agent.git` |
| Binary | `~/.local/bin/hermes` (on `PATH`) |
| Config | `~/.hermes/config.yaml`, `~/.hermes/.env` |
| Sessions / skills | `~/.hermes/` |
| Python | 3.14.x |
| Model | `deepseek/deepseek-v4.1-flash-1017` via OpenRouter |
| Gateway | Installed; stopped at the time of setup |

Check it:

```bash
hermes --version
hermes status
hermes doctor
```

### 3.2 Update safely

Back up the important state first (config, env, auth):

```bash
mkdir -p /tmp/opencode/hermes-backup
cp -a ~/.hermes/config.yaml ~/.hermes/.env ~/.hermes/auth.json \
  /tmp/opencode/hermes-backup/
```

Stale lock files from an interrupted run were blocking updates. Remove them only when
no Hermes process is running:

```bash
pgrep -af hermes
rm -f ~/.hermes/.hermes-update-in-progress.lock
```

Check and update:

```bash
hermes update --check
hermes update --yes
```

Outcome on this machine:

- Already on the latest **stable** release: `v0.23.4 (8fc0374)`.
- Upstream `main` had newer commits, but stable correctly stayed put.
- The update cleared 110 stale `__pycache__` directories, refreshed Node deps,
  TUI, Web UI, model catalog cache, and bundled skills.

To move to bleeding edge (optional, not done):

```bash
hermes update --set-channel   # or: hermes update --branch main
```

### 3.3 Fix: `cua-driver` / missing `libXi.so.6`

The optional CUA driver failed because `libXi.so.6` was missing. Fix:

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y libxi6
hermes pm install cua-driver
hermes pm doctor
```

Result: `cua-driver` installs cleanly, and `hermes pm doctor` shows no failures
(optional runtimes such as llama.cpp / Whisper simply are not installed).

### 3.4 Using Hermes — the short version

Start:

```bash
hermes                          # classic interactive chat
hermes --tui                    # modern full-screen interface
hermes -c                       # resume the most recent session
hermes -c "session name"        # resume a named session
hermes -z "prompt"              # one-shot, print answer and exit
hermes chat -q "..." --image pic.png
```

Inside a session:

| Command | What it does |
|---|---|
| `/help` | All slash commands |
| `/status` | Model, tokens, cwd, session recap |
| `/context` | What is filling the context window |
| `/model` | Show/switch model (`--global` to persist) |
| `/tools`, `/toolsets` | See what the agent can do |
| `/approvals` | `manual`, `smart`, or `off` |
| `/reasoning high` | More thinking effort |
| `/title My task` | Name the session |
| `/new` | Fresh session |
| `/undo` | Remove the last exchange |
| `/compress` | Summarize old context |
| `/save md notes.md` | Export the conversation |
| `/bg <prompt>` | Run a task in a background session |
| `/btw <question>` | Side question without derailing the turn |
| `/skills`, `/learn <source>` | Browse skills; turn a workflow into a skill |
| `/init` | Generate `AGENTS.md` for the current repo |
| `/cron` | Manage scheduled jobs |
| `!git status` | Run a shell command yourself, no model turn |

**Memory and reusable behavior**

- `~/.hermes/memories/` — what Hermes remembers about you.
- `AGENTS.md` in a project — rules read every session; create with `/init`.
- `~/.hermes/SOUL.md` — persona / tone.
- `~/.hermes/skills/` — installed skills (39 built in, e.g. `pdf`, `github`,
  `systematic-debugging`, `google-workspace`).

Say *“remember that I deploy with `make release`”* to persist a fact, and
*“save this as a skill”* after a workflow you want to repeat.

**Sessions and skills**

```bash
hermes sessions list
hermes sessions export <id> --format md
hermes skills list
hermes skills search pdf
hermes skills install <name>
hermes -s pdf "summarize report.pdf"
```

**Gateway (Telegram / Discord / Slack / WhatsApp / Signal)**

```bash
hermes gateway status
hermes gateway setup      # connect a platform; needs a bot token
hermes gateway install    # survive reboots as a service
```

Then DM the bot and approve the pairing code on the machine.
`/handoff telegram` moves a live CLI session to the phone.

**Automation**

```bash
hermes cron list
hermes cron add
hermes cron status
```

Cron jobs run in **fresh** sessions, so prompts must be fully self-contained. It is
usually easier to just ask in chat: *“Every weekday at 8am, check my GitHub
notifications and send a summary on Telegram.”*

**Maintenance**

```bash
hermes update
hermes doctor
hermes doctor --fix
hermes backup
hermes logs
hermes --safe-mode     # troubleshooting: ignore all customizations
```

A fuller command reference is saved at **`~/Notes/hermes-cheatsheet.md`**.

> At setup time the Hermes gateway was installed but had no platforms connected.
> Run `hermes gateway setup` when you want phone access.

---

## 4. Free Claude Code + Claude Code

### 4.1 What this is

[Free Claude Code](https://github.com/Alishahryar1/free-claude-code) is a local proxy
that exposes an **Anthropic-compatible API** and routes Claude Code requests to
OpenRouter models — in this setup, free ones. Claude Code connects to
`http://127.0.0.1:8082`; the proxy translates and forwards the requests.

### 4.2 Install

The upstream installer is interactive. It was driven over a PTY so only **Claude Code**
was selected; Codex, Pi, Cline, DeepSeek Harness, Grok, Muse, Aider, Antigravity, and
the RTK/Headroom optimizer were declined.

```bash
cd /tmp/opencode
curl -fsSL https://raw.githubusercontent.com/Alishahryar1/free-claude-code/main/scripts/install.sh \
  -o fcc-install.sh

# Run it interactively (choose "Claude Code" only when prompted)
sh fcc-install.sh
```

Installed components:

| Component | Version / path |
|---|---|
| FCC proxy | **6.19.56**, `~/.local/bin/fcc-server` |
| Claude Code CLI | **2.1.332**, `~/.local/bin/claude` |
| Config | `~/.fcc/.env` (mode `600`) |
| Database | `~/.fcc/fcc.db` |
| Logs | `~/.fcc/logs/server.log` |
| Proxy | `http://127.0.0.1:8082` (Anthropic + OpenAI compatible) |
| Admin UI | `http://127.0.0.1:8082/admin` |

### 4.3 Configuration

The proxy was bound to loopback only. This matters: FCC's `/v1` API has no auth by
default, so exposing `0.0.0.0:8082` on a VPS would let anyone who can reach the port
use the OpenRouter key.

| Setting | Value | Why |
|---|---|---|
| `MODEL` | `open_router/nvidia/nemotron-3-ultra-550b-a55b:free` | strong free coding model |
| `MODEL_HAIKU` | `open_router/nvidia/nemotron-3.5-lightning:free` | cheaper/faster background tier |
| `MODEL_FALLBACKS` | `nemotron-3.5-lightning:free` → `openrouter/free` | keeps working through free-tier rate limits |
| `HOST` | `127.0.0.1` | no unauthenticated proxy on the public network |

Sonnet/Opus tiers inherit the default, so real coding turns use the 550B model.

Change any of these through the Admin UI, or via the admin API
(`http://127.0.0.1:8082/admin/api/config`).

### 4.4 Run as a service (survives logout and reboot)

Unit file: `~/.config/systemd/user/fcc-server.service`

```ini
[Unit]
Description=Free Claude Code proxy (localhost Anthropic-compatible API)
Documentation=https://github.com/Alishahryar1/free-claude-code
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/home/ubuntu/.local/bin/fcc-server
Environment=PATH=/home/ubuntu/.local/bin:/usr/local/bin:/usr/bin:/bin
Environment=HOME=/home/ubuntu
EnvironmentFile=/home/ubuntu/.secrets/openrouter.env
Restart=on-failure
RestartSec=5
TimeoutStopSec=20

[Install]
WantedBy=default.target
```

Enable boot-start even without an SSH session, then start it:

```bash
sudo loginctl enable-linger ubuntu
systemctl --user daemon-reload
systemctl --user enable --now fcc-server.service
```

Check:

```bash
systemctl --user is-active fcc-server
systemctl --user is-enabled fcc-server
curl -s http://127.0.0.1:8082/health
journalctl --user -u fcc-server -f
```

### 4.5 Using it

Always launch **`fcc-claude`**, not bare `claude` — `fcc-claude` injects the proxy
URL and auth token.

```bash
cd ~/testingvps
fcc-claude

# one-shot
fcc-claude -p "Reply with exactly: OK"
```

Other commands:

```bash
fcc-server --version
fcc-doctor                 # diagnostics; expect "errors": []
```

Remote Admin UI from your laptop:

```bash
ssh -L 8082:127.0.0.1:8082 ubuntu@<this-host>
# then open http://127.0.0.1:8082/admin
```

### 4.6 Caveats

- The free models have **per-minute rate limits**. The fallback chain is there for
  that; if you see frequent `429`s, point `MODEL` at a cheap paid model
  (DeepSeek V4.1 Flash or GLM-5.3) for reliability.
- Boot-start was enabled but the box was not physically rebooted during setup.
  A reboot test is worthwhile.
- One FCC instance only. The ad-hoc server was stopped so only the systemd service
  owns port `8082`.

---

## 5. Single-source API key

### 5.1 The problem

The OpenRouter key originally existed in three files:

| Path | Role |
|---|---|
| `~/.fcc/.env` | FCC's own store |
| `~/.hermes/.env` | original source used by Hermes |
| `~/.hermes/state-snapshots/.../.env` | copy made by Hermes's pre-update backup |

Symlinking the `.env` files is not a good fix: each file contains different settings,
and both apps rewrite their own file.

### 5.2 The solution

One secret file, mode `600`, in a `700` directory:

```bash
umask 077
mkdir -p ~/.secrets
# create ~/.secrets/openrouter.env containing:
# OPENROUTER_API_KEY=<your key>
chmod 600 ~/.secrets/openrouter.env
```

Both consumers now read from it:

| Consumer | Mechanism |
|---|---|
| **FCC** | `EnvironmentFile=/home/ubuntu/.secrets/openrouter.env` in the systemd unit |
| **Hermes** | `~/.profile` and `~/.bashrc` source the file into the environment |

The sourcing block appended to both shell files:

```bash
# Single-source local secrets (e.g. OPENROUTER_API_KEY for Hermes/FCC)
if [ -r "$HOME/.secrets/openrouter.env" ]; then
    set -a
    . "$HOME/.secrets/openrouter.env"
    set +a
fi
```

Cleanup that was performed:

- Removed `OPENROUTER_API_KEY=` from `~/.fcc/.env`.
- Removed `OPENROUTER_API_KEY=` from `~/.hermes/.env`.
- Deleted the pre-update snapshot's `.env` (kept the rest of the snapshot for rollback).
- Deleted a temporary copy made under `/tmp/opencode/hermes-backup/`.
- Removed a bad `"model"` override from `~/.claude/settings.json` that caused an
  `unrecognized_model` warning.

### 5.3 Verify

Only one file should contain the key:

```bash
grep -rl '^OPENROUTER_API_KEY=' ~/.secrets ~/.fcc ~/.hermes 2>/dev/null
```

Check that each consumer sees it:

```bash
# Shell / Hermes
bash -lc 'test -n "$OPENROUTER_API_KEY" && echo key-present'

# FCC service process
pid=$(systemctl --user show -p MainPID --value fcc-server)
tr '\0' '\n' < /proc/$pid/environ | grep -c '^OPENROUTER_API_KEY=.'
```

FCC's admin API should report the credential as `source=process`, `locked=true`, and
`~/.fcc/.env` should have zero key lines.

### 5.4 Caveat

The key now arrives through the environment, so it depends on the launcher:

- `systemctl --user ... fcc-server` → works (`EnvironmentFile`).
- `fcc-claude` from a normal login/interactive shell → works (`.profile` / `.bashrc`).
- A future daemon that skips both (e.g. a Hermes gateway service) would not see it —
  add `EnvironmentFile=` to that unit.

Alternative with zero shell-init dependence: a root-owned system service that also uses
`EnvironmentFile`.

---

## 6. End-to-end verification checklist

Run these after any change:

```bash
# GitHub
gh --version
gh auth status
git -C ~/testingvps remote -v

# Hermes
hermes --version
hermes status
hermes -z "Reply with exactly: HERMES-OK"

# FCC / Claude Code
systemctl --user is-active fcc-server
curl -s http://127.0.0.1:8082/health
fcc-doctor
fcc-claude -p "Reply with exactly: FCC-OK"

# Secrets
stat -c '%A %U %n' ~/.secrets/openrouter.env
grep -rl '^OPENROUTER_API_KEY=' ~/.secrets ~/.fcc ~/.hermes 2>/dev/null
```

Everything is healthy when both one-shot replies come back and `fcc-doctor` reports
`"errors": []`.

---

## 7. Day-to-day quick reference

| Task | Command |
|---|---|
| Start Hermes TUI | `hermes --tui` |
| Resume Hermes session | `hermes -c` |
| Ask Hermes one-shot | `hermes -z "question"` |
| Install/update a Hermes skill | `hermes skills install <name>` |
| Check Hermes health | `hermes doctor` |
| Start free Claude Code | `cd ~/testingvps && fcc-claude` |
| Check proxy | `curl -s http://127.0.0.1:8082/health` |
| Restart proxy | `systemctl --user restart fcc-server` |
| Proxy logs | `journalctl --user -u fcc-server -f` |
| Diagnose FCC | `fcc-doctor` |
| Admin UI (remote) | `ssh -L 8082:127.0.0.1:8082 ubuntu@<host>` |
| Create a private repo | `gh repo create <name> --private` |
| Clone a repo | `gh repo clone <owner>/<repo> <folder>` |
| Check key location | `grep -rl '^OPENROUTER_API_KEY=' ~/.secrets ~/.fcc ~/.hermes` |

---

## 8. Troubleshooting and gotchas

| Symptom | Cause / fix |
|---|---|
| `gh` is not authenticated | Re-run `gh auth login --web`; approve the device code on any browser |
| `gh auth login` waits forever | It is waiting for browser approval; check `gh auth status` |
| `hermes update` blocked | Stale `.hermes-update-in-progress.lock`; confirm no Hermes process, then remove it |
| `cua-driver` install fails | `sudo apt-get install -y libxi6`, then retry `hermes pm install cua-driver` |
| `unrecognized_model` in Claude Code | Remove the `"model"` override from `~/.claude/settings.json` |
| `429` from FCC | Free-tier rate limit; fallbacks should catch it, or switch `MODEL` to a paid model |
| `fcc-claude` works but a daemon does not see the key | That unit needs `EnvironmentFile=/home/ubuntu/.secrets/openrouter.env` |
| Port `8082` reachable from outside | Set `HOST=127.0.0.1` in FCC config and restart the service |
| `origin/main [gone]` after clone | Normal for an empty repo; it resolves after the first push |

**Security rules of thumb**

- Never commit `~/.secrets/`, `~/.fcc/.env`, `~/.hermes/.env`, or
  `~/.config/gh/hosts.yml`.
- Keep the FCC proxy on `127.0.0.1`; use an SSH tunnel for the Admin UI.
- Keep secret files at mode `600` and `~/.secrets` at `700`.
- Prefer the device-code flow over pasting tokens into a chat.

---

## Appendix — Source sessions

This guide merges the content of three OpenCode sessions from **26 Nov 2026**:

1. **“Assistant identity question”** (13:22–13:29) — installed GitHub CLI and started
   the device-code login.
2. **“gh installation availability”** (13:33–14:06) — authenticated `gh`, created and
   cloned `testingvps`, built the home-folder layout, updated Hermes, installed
   Free Claude Code, set up its systemd service, and consolidated the API key.
3. **“Learning to use Hermes”** (14:11–14:12) — practical Hermes usage guide and the
   saved cheat sheet at `~/Notes/hermes-cheatsheet.md`.
