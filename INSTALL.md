# INSTALL — signal-agent-pipeline

Last install-tested (up to, not including, the phone-linking step) on 2026-10-06
with signal-cli 0.14.7 + Java 25 (Temurin 25) on Debian 13 (x86_64).
Steps marked **[owner]** need a human with the phone in hand; everything else
can be run by an agent.

## 0. Dependencies

| what | why | install |
|---|---|---|
| **Java 25** (signal-cli ≥ 0.14.5 is built for Java 25; Java 21 fails with `UnsupportedClassVersionError`) | `signal-cli` runtime | Debian 13 / Ubuntu with openjdk-25 available: `sudo apt install openjdk-25-jre-headless`. Otherwise unpack a Temurin 25 JRE (https://adoptium.net) anywhere, e.g. `/opt/jdk25`, and set `JAVA_HOME_SIGNAL_CLI` to that directory |
| `signal-cli` **0.14.7** (skip 0.14.8: its `listDevices` panics and hangs, upstream issue AsamK/signal-cli#2119; any later release with that fix is fine) | the Signal client | `signal-cli-0.14.7.tar.gz` from https://github.com/AsamK/signal-cli/releases/tag/v0.14.7 → unpack to `/opt/signal-cli` (or anywhere; set `SIGNAL_CLI_BIN`) |
| `python3` | JSON result parsing | present on both OSes |
| `rclone` (optional) | off-box ledger backup | `sudo apt install rclone` |
| `proxychains4` + a SOCKS5 proxy (optional) | only if Signal endpoints are blocked on this network | `sudo apt install proxychains4` / `brew install proxychains-ng` |

## 1. Put the scripts on PATH and set the environment

```bash
git clone https://github.com/starshard-ai/signal-agent-pipeline
cd signal-agent-pipeline
mkdir -p ~/bin && cp bin/signal-agent bin/fleet-drop ~/bin/ && chmod +x ~/bin/signal-agent ~/bin/fleet-drop
mkdir -p ~/.config && cp .env.example ~/.config/signal-agent.env   # then edit it
```

`~/bin` is only added to your PATH by a new login shell (on Debian/Ubuntu,
`~/.profile` adds it if it exists). **Open a new terminal** now, or run
`export PATH="$HOME/bin:$PATH"` in this one.

Set at least `SIGNAL_CLI_BIN`, `SIGNAL_DEVICE_NAME`, and — unless a Java 25
`java` is already first on your PATH — `JAVA_HOME_SIGNAL_CLI`. You do **not**
need to source the file from `~/.bashrc`: `bin/signal-agent` loads
`~/.config/signal-agent.env` itself, so systemd units, cron and
`ssh box signal-agent …` get the same settings. Variables already set in your
environment take precedence over the file. The real file is **never** committed.

Check the setup before linking (nothing here talks to your phone or account):

```bash
signal-agent whoami    # before linking: prints [] (no account yet). Java or path problems show up here.
signal-agent recent    # "no messages yet" before the first receive
```

## 2. Link the box as a secondary device  **[owner]**

```bash
signal-agent link        # uses SIGNAL_DEVICE_NAME from ~/.config/signal-agent.env
```

It prints a `sgnl://linkdevice?...` URI and a QR code and blocks. On the phone:
Signal → Settings → Linked Devices → “+” → scan. When the command returns, run:

```bash
signal-agent whoami      # your device name must be listed
signal-agent receive     # first sync; may take ~1 min; prints [] when nothing is queued
```

If `receive` prints Java class-version errors, your Java is too old for this
signal-cli build: install Java 25 and point `JAVA_HOME_SIGNAL_CLI` at it.

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

Units use `%h` (your home) and call `~/bin/signal-agent`, which loads
`~/.config/signal-agent.env` itself, so there is nothing to edit — with one
exception: the `.path` unit watches the **default** ledger
`~/.signal-agent/messages.jsonl`. If you changed `SIGNAL_HOME`, edit its
`PathModified=` line. The timer runs `receive` hourly; the `.path` unit runs
`backup` whenever `messages.jsonl` changes (while it is active, `receive` skips
its own inline backup so nothing runs twice). `backup` is a no-op until `rclone`
has a remote named `$SIGNAL_RCLONE_REMOTE`.

On macOS use `launchd` (a `StartInterval` plist calling `~/bin/signal-agent receive`)
or simply run `receive` on demand from your agent.

## 5. Blocked network? (optional)

If `signal-cli` cannot reach Signal directly:

```bash
cp examples/proxychains.conf.example ~/.config/signal-agent-proxychains.conf   # set your SOCKS5 host:port
```

Then add to `~/.config/signal-agent.env` (keep `SIGNAL_CLI_BIN` pointing at
signal-cli itself; the wrapper goes in a separate prefix variable):

```bash
SIGNAL_CLI_PREFIX="proxychains4 -q -f /home/you/.config/signal-agent-proxychains.conf"
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
| `UnsupportedClassVersionError` | Java too old for this signal-cli build (needs 25) → set `JAVA_HOME_SIGNAL_CLI` to a Java 25 directory |
| `JAVA_HOME is set to an invalid directory` | `JAVA_HOME` / `JAVA_HOME_SIGNAL_CLI` points at a folder without `bin/java` → fix the path, or unset it and put a Java 25 `java` first on PATH |
| `whoami` / `listDevices` hangs, `libsignal-tokio-worker panicked` | signal-cli 0.14.8 bug (AsamK/signal-cli#2119) → use 0.14.7 or a later fixed release |
| `SIGNAL_CLI_BIN must be the path to signal-cli only` | an old proxychains recipe put the proxy command into `SIGNAL_CLI_BIN` → move it to `SIGNAL_CLI_PREFIX` (step 5) |
| `send` prints `SEND FAILED — no parseable JSON` | bad recipient or signal-cli crashed; run the same `signal-cli` command by hand to see the error |
| `receive` returns nothing for a long time | linked devices only get messages sent *after* linking; also check the phone is online once |
| Chinese/emoji arrive as `???` | locale: the wrapper already exports `LC_ALL=C.UTF-8` and `-Dfile.encoding=UTF-8`; make sure your unit/cron inherits it |
| `fleet-drop: route down` | ssh key not on the relay box, or Tailscale/VPN down → try `FLEET_DROP_HOST_FALLBACK` (public IP) |
| server `ACCEPTED` but recipient never sees it | acceptance ≠ delivery; recipient device offline. Do **not** resend blindly — check `signal-agent receive` for receipts first |
