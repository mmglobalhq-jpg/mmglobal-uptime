# mmglobal-uptime

An off-box liveness check for the MMGlobal platform. One workflow, no secrets,
nothing else.

## Why this repo exists at all

The platform is self-hosted on a single mini PC. On **2026-08-12** a Windows
Update reboot left that machine sitting at the login screen, and it stayed there
for **15 hours 54 minutes** — until a person walked over and typed a password.

It had monitoring. The monitoring did not help, because all of it ran on the
machine that was off: an on-host watchdog timer, a heartbeat service, and the
alerting path those used. **A monitor that only runs when the thing it monitors
is healthy is not a monitor.**

So this check lives on GitHub's infrastructure instead. It survives the failure
it is watching for. That property is the whole design; please don't optimise it
away by folding this back into the host.

## What it checks

Every ~10 minutes, three public endpoints, three attempts each before it will
call anything down:

| Host | Expected | Meaning |
|---|---|---|
| `chat.mmglobal.us` | `200` | The full chain is alive: Windows logon → WSL → systemd → Docker → cloudflared |
| `api.mmglobal.us` | `302` | Cloudflare Access is still enforcing — a `200` here means the API got exposed |
| `desktop.mmglobal.us` | `200` | Guacamole is serving |

The retries matter: cloudflared reconnects and Cloudflare edge blips each produce
a single bad response, and alerting on one sample would cry wolf constantly.

## How alerting works

There is none configured, deliberately — the workflow simply **fails**, and
GitHub emails the account owner when a scheduled workflow fails. No credentials
are stored in this repo, which is why it can safely be public.

## Honest limitations

- **GitHub's cron is best-effort.** A `*/10` schedule commonly lands 5–15 minutes
  late and occasionally skips a slot. Real detection time is "within ~30 minutes",
  not ten. That is a poor pager and an excellent smoke alarm; the gap it was built
  to close was sixteen hours.
- **It only sees the front door.** A green run means the edge answers. It says
  nothing about whether the fund pollers ingested, whether backups ran, or whether
  the deployed image is the intended one — all of which have failed while the edge
  returned `200`. The on-host `platform-watchdog.timer` covers that layer, and the
  two are complements, not substitutes.
- **This repo is public, so the workflow file and its run logs are public.** They
  contain hostnames that are already public DNS records, and HTTP status codes.
  Nothing else. Keep it that way — do not add a secret here.

The private write-up of the incident and the recovery procedure lives in
`mmglobalhq-jpg/system-source-of-truth`, `docs/21-disaster-recovery.md` §2.
