# signal-agent-pipeline

Give a personal AI agent a **Signal channel** without a bot number: `signal-cli`
runs as a **linked secondary device** on your own Signal account (exactly like
Signal Desktop), and a thin shell wrapper turns it into four verbs an agent can
call — `receive`, `send`, `send-attach`, `send-group` — plus a file-drop route
from any other machine you own to your phone.

Part of the Starshard communication stack
([starshard-communication](https://github.com/starshard-ai/starshard-communication)
is the protocol layer; this repo is one **channel adapter**).

## What is in the box

| path | role |
|---|---|
| `bin/signal-agent` | wrapper over `signal-cli`: `link`, `whoami`, `receive`, `recent`, `send`, `send-attach`, `send-group`, `groups`, `contacts`, `backup` |
| `bin/fleet-drop` | from any machine: `scp` a file to the linked box, then `signal-agent send-attach --note-to-self` → lands in Signal *Note to Self* on your phone |
| `systemd/` | user units: hourly `receive` timer (fail-safe cadence) + a `.path` unit that backs up the ledger the moment it changes (event-driven primary) |
| `examples/proxychains.conf.example` | only needed when your network blocks Signal's endpoints |
| `.env.example` | every knob; nothing secret lives in the repo |

## Design decisions (why it looks like this)

- **Linked device, not a bot.** The agent reads and writes *your* conversations
  with the people you choose; there is no second identity to manage, and
  end-to-end encryption is untouched.
- **Proof of send = server result, not exit code.** `send` parses `signal-cli -o json`
  `results[].type` and only prints a timestamp when the server said `SUCCESS`.
  `ACCEPTED` by the server is still not proof of delivery or reading; treat it that way.
- **Flat-file ledgers.** `messages.jsonl` (received) and `sent.jsonl` (sent) under
  `$SIGNAL_HOME`; `cat` to read, `rm` to forget. No database.
- **Event-driven first, timer as fail-safe.** Backup fires on file change; the
  hourly `receive` timer only bounds worst-case staleness.
- **Clean egress host.** If you sit behind a firewall that blocks Signal, run the
  linked device on a small VPS with clean egress and reach it over ssh — that is
  what `fleet-drop` assumes. Otherwise run everything on one box.

## Quick start

See [INSTALL.md](INSTALL.md). The only owner-gated step is scanning the link QR
from Signal on your phone; everything else is scriptable.

```bash
signal-agent link my-agent-box     # prints a sgnl:// URI + QR — scan it from Signal > Linked Devices
signal-agent whoami                # lists devices; your new one should appear
signal-agent receive               # pulls whatever is queued into $SIGNAL_HOME/messages.jsonl
signal-agent send +15551234567 "hello from my agent"
signal-agent send-attach --note-to-self ./report.pdf "tonight's report"
fleet-drop --note "draft pack" ./draft.md ./fig1.png   # from another machine
```

## Security notes

- The linked device holds full read/write on your account. Keep the box's
  `~/.local/share/signal-cli` directory private (`chmod 700`) and off any shared
  backup that others can read.
- Nothing in this repo stores or transmits credentials; `.env.example` documents
  the environment variables and the real values stay in your shell profile.
- Unlink at any time from your phone (Signal > Linked Devices); the box loses
  access immediately.

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).
