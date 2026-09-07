# INSTALL — signal-agent-pipeline

Tested on Ubuntu 24.04 (VPS) and macOS 14+. Steps marked **[owner]** need a human
with the phone in hand; everything else can be run by an agent.

## 0. Dependencies

| what | why | install |
|---|---|---|
| Java 21+ (signal-cli 0.14.x wants 21; some builds want 25) | `signal-cli` runtime | `sudo apt install openjdk-21-jre` — or unpack a Temurin JDK to `/opt/jdk25` and set `JAVA_HOME_SIGNAL_CLI` |
| `signal-cli` ≥ 0.13 | the Signal client | release tarball from https://github.com/AsamK/signal-cli/releases → unpack to `/opt/signal-cli` |
| `python3` | JSON result parsing | present on both OSes |
| `rclone` (optional) | off-box ledger backup | `sudo apt install rclone` |
| `proxychains4` + a SOCKS5 proxy (optional) | only if Signal endpoints are blocked on this network | `sudo apt install proxychains4` / `brew install proxychains-ng` |

## 1. Put the scripts on PATH and set the environment

```bash
git clone https://github.com/starshard-ai/signal-agent-pipeline
cd signal-agent-pipeline
mkdir -p ~/bin && cp bin/signal-agent bin/fleet-drop ~/bin/ && chmod +x ~/bin/signal-agent ~/bin/fleet-drop
cp .env.example ~/.config/signal-agent.env   # edit it; then add to ~/.bashrc:
#   set -a; . ~/.config/signal-agent.env; set +a
```

Minimum you must set: `SIGNAL_CLI_BIN`, `SIGNAL_HOME`, `SIGNAL_DEVICE_NAME`.
The real `.env` is **never** committed (`.gitignore` already excludes it).

## 2. Link the box as a secondary device  **[owner]**

```bash
signal-agent link "$SIGNAL_DEVICE_NAME"
```

It prints a `sgnl://linkdevice?...` URI and a QR code and blocks. On the phone:
Signal → Settings → Linked Devices → “+” → scan. When the command returns, run:

```bash
signal-agent whoami      # your device name must be listed
signal-agent receive     # first sync; may take ~1 min; prints [] when nothing is queued
```

If `receive` prints Java class-version errors, your system Java is too old for
this signal-cli build: install the matching JDK and point `JAVA_HOME_SIGNAL_CLI` at it.

## 3. Send something

```bash
signal-agent send --note-to-self "linked and working"
signal-agent send +<E164 number> "hi"                # or a recipient UUID from `contacts`
signal-agent send-attach --note-to-self ./file.pdf "caption"
```

`send` exits 0 **only** when the server returned `SUCCESS` for every recipient
and prints the server timestamp. Non-zero = not sent; do not assume delivery.

## 4. Schedule receive + backup (systemd user units, Linux)

```bash
mkdir -p ~/.config/systemd/user
cp systemd/* ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now signal-agent-receive.timer signal-agent-backup.path
systemctl --user list-timers | grep signal
loginctl enable-linger "$USER"      # keep user units alive without a login session
```

Units use `%h` (your home) — no paths to edit. The timer runs `receive` hourly;
the `.path` unit runs `backup` whenever `messages.jsonl` changes. `backup` is a
no-op until `rclone` has a remote named `$SIGNAL_RCLONE_REMOTE`.

On macOS use `launchd` (a `StartInterval` plist calling `~/bin/signal-agent receive`)
or simply run `receive` on demand from your agent.

## 5. Blocked network? (optional)

If `signal-cli` cannot reach Signal directly:

```bash
cp examples/proxychains.conf.example ~/.config/signal-agent-proxychains.conf   # set your SOCKS5 host:port
export SIGNAL_CLI_BIN="proxychains4 -q -f ~/.config/signal-agent-proxychains.conf /opt/signal-cli/bin/signal-cli"
```

The recommended alternative is a small VPS with clean egress as the linked
device, reached over ssh — which is what `fleet-drop` is for.

## 6. fleet-drop (files from any machine → your phone)

On the *other* machine (laptop, another server):

```bash
export FLEET_DROP_HOST=user@relay-host            # the box from steps 1–3, ssh key auth
export FLEET_DROP_HOST_FALLBACK=user@relay-public-ip   # optional second route
fleet-drop --check                                # prints device list + "route OK via <host>"
fleet-drop --note "tonight's draft" ./draft.md ./fig.png
```

Exit 0 only when every file was `ACCEPTED` by the Signal server. 95 MB per-file cap.

## 7. Token / secret management

- There are **no API tokens** in this pipeline; the only credential is the
  linked-device key material that `signal-cli` stores under
  `~/.local/share/signal-cli/` — `chmod 700` it and keep it out of shared backups.
- ssh between machines uses keys, never passwords; `fleet-drop` runs with
  `BatchMode=yes` so it can never prompt.
- If you ever suspect the box is compromised: phone → Linked Devices → remove it.
  Access ends immediately, no other rotation needed.

## Common failures

| symptom | cause / fix |
|---|---|
| `UnsupportedClassVersionError` | Java too old for this signal-cli build → set `JAVA_HOME_SIGNAL_CLI` to a newer JDK |
| `send` prints `SEND FAILED — no parseable JSON` | bad recipient or signal-cli crashed; run the same `signal-cli` command by hand to see the error |
| `receive` returns nothing for a long time | linked devices only get messages sent *after* linking; also check the phone is online once |
| Chinese/emoji arrive as `???` | locale: the wrapper already exports `LC_ALL=C.UTF-8` and `-Dfile.encoding=UTF-8`; make sure your unit/cron inherits it |
| `fleet-drop: route down` | ssh key not on the relay box, or Tailscale/VPN down → try `FLEET_DROP_HOST_FALLBACK` (public IP) |
| server `ACCEPTED` but recipient never sees it | acceptance ≠ delivery; recipient device offline. Do **not** resend blindly — check `signal-agent receive` for receipts first |
