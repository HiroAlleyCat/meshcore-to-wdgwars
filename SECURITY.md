# Security Notes

## What this tool does

- Reads a local CSV or JSON file (your MeshMapper export or MeshCore
  offline ping log).
- Normalises each row to the WDGWars meshcore schema.
- Optionally POSTs the records to `https://wdgwars.pl/api/upload/` as
  an HMAC-signed JSON envelope.
- On request only, checks GitHub's releases API for a newer version
  (`--check-version`) and self-updates (`--update`). See below.

## Outbound network footprint

**Nothing here contacts anybody unless you invoked a command that says
so.** There is no background check, no timer, and no call on an ordinary
run.

- **Uploads** go only to the configured WDGWars endpoint
  (`https://wdgwars.pl/api/upload/` by default, override with
  `--api-url`), and only when you invoke an upload without `--dry-run`.
- **Version check** (`--check-version`): queries GitHub's releases API
  and prints whether a newer release exists. 3 s timeout. It is worth
  knowing what that request shows GitHub, since GitHub is a third party
  and not us: your source IP, the exact version you are running (it is
  in the User-Agent), which tool you are running, and the time you ran
  it. That is why it is a command you type rather than something this
  tool does on its own. Earlier releases checked once per 24 h on almost
  every run, with `--no-version-check` and `--quiet` as the opt-out.
  That was the wrong default and it is gone. The flag is still accepted
  and ignored so existing cron lines and scheduled tasks keep working.
- **Self-update** (`--update`, or the `update.sh`/`update.bat`
  wrappers): fetches `heimdall.py`, `requirements.txt`, and the wrapper
  scripts from this repo's `main` branch on raw.githubusercontent.com
  over HTTPS. The downloaded script is AST-parsed before it atomically
  replaces the old one. This is an explicit, operator-invoked action.
  Nothing updates itself in the background.
- **No telemetry or analytics.** Nothing else leaves the machine.
- ❌ **No `eval`, `exec`, `os.system`, or `shell=True` subprocess calls.**
  Subprocesses (git/pip/systemctl/crontab/schtasks in `--update` and the
  scheduler installers) run with argument lists, never through a shell.

## API key handling

- Resolution: `--key` flag → `$WDGWARS_API_KEY` env var → saved key file
  (`~/.config/heimdall/api.key`, `%APPDATA%\heimdall\api.key` on
  Windows). `--api-key` survives only as a hidden deprecated alias.
- `--setup` / `--save-key` persist the key with `mode 0600`, an atomic
  create, and symlink refusal, same shape as Muninn's
  `~/.config/muninn/api.key`.
- The key never appears in scheduler unit files, cron lines, or task
  definitions. Scheduled runs re-read the saved key file at run time.
- The key is sent over HTTPS only, in the `X-API-Key` request header
  to `wdgwars.pl`. The TLS context is Python's `ssl.create_default_context()`
  default, system trust store, hostname verification on, TLS 1.2+.
- Heimdall does not print the API key in any output, success or
  failure. If it ever shows up in a server-response body or stack
  trace, that's a bug, please open an issue.

## What the API key can do

The WDGWars API key authorises you to submit observations under your
account. If it leaks, an attacker could:

- Submit fake mesh / WiFi / BLE / aircraft captures under your name.
- Read your account stats via `GET /api/me`.

It cannot (as far as we know):

- Change your password.
- Withdraw money / make purchases.
- Affect other users' accounts.

If you suspect your key has leaked, rotate it on the WDGWars site and
re-run Heimdall with the new key.

## HMAC envelope

The envelope shape is byte-identical to Muninn's:

```python
payload   = {"networks": [], "aircraft": [], "meshcore_nodes": chunk}
body_json = json.dumps(payload, separators=(",", ":"))
data_b64  = base64.b64encode(body_json.encode()).decode()
nonce     = secrets.token_hex(8)
sig       = hmac.new(api_key.encode(),
                     (nonce + data_b64).encode(),
                     hashlib.sha256).hexdigest()
envelope  = {"data": data_b64, "nonce": nonce, "sig": sig}
```

`json.dumps(..., separators=(",", ":"))` and `ensure_ascii=True`
(Python's default) are load-bearing, different whitespace or
non-ASCII handling produces a different signature.

## Static-analysis review

A review of Heimdall against the SonarCloud SAST finding classes (path
traversal, command/argument injection, insecure temp use, unsafe DB opens)
found nothing to remediate, the scheduler arguments (including the CSV path)
are shell-quoted, the API key never reaches the command line, and `save_key`
refuses symlinks and uses mode 600. The full write-up is in
[SECURITY-FINDINGS.md](SECURITY-FINDINGS.md); the posture is locked by
`tests/test_security.py`.

## Reporting issues

Open a GitHub issue, or DM the maintainer on the WDGWars community
channels. For anything potentially exploitable upstream (in WDGWars
itself), please disclose privately to LOCOSP first rather than filing
a public issue here.
