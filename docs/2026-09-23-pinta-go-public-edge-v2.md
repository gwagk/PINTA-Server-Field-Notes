# PINTA GO — Public Edge & V2 Field Notes

**Date:** 23 September 2026  
**Host:** PINTA Server  
**Principle:** Lean, safe, reversible. PINTA is a lab/utility server, not a data center.

## What we proved

PINTA GO can leave the private lab and provide a public web utility without exposing SSH or requiring inbound router port forwarding.

Public service path:

```text
Internet
  -> HTTPS / TCP 443
Cloudflare Edge
  -> Cloudflare Tunnel (outbound from PINTA; observed QUIC)
cloudflared on PINTA
  -> HTTP / TCP 80
Nginx
  -> HTTP / TCP 5000 on 127.0.0.1
Gunicorn
  -> Flask / PINTA GO
```

Administration remains a separate plane:

```text
JETEK / Termius
  -> Tailscale encrypted overlay
  -> SSH / TCP 22
  -> PINTA
```

Do not confuse these paths: Cloudflare Tunnel does not run through Tailscale.

## Cloudflare Quick Tunnel PoC

`cloudflared 2026.9.1` was installed and a Quick Tunnel was tested successfully.

```bash
cloudflared tunnel --url http://localhost:80
```

The temporary `*.trycloudflare.com` address successfully exposed PINTA GO through Cloudflare -> Nginx -> Gunicorn. Short URL creation and QR generation worked from the public side.

Important field lesson: Quick Tunnel exists only while the foreground `cloudflared` process is alive. Pressing `Ctrl+C` terminates it. In Termius/Linux, use `Ctrl+Shift+C` for copy.

A permanent public hostname is a later step: registered domain/Cloudflare zone + managed/named tunnel + service startup. Do not make PINTA an authoritative DNS server merely to avoid this; that adds responsibility without solving the public-domain problem.

## PINTA GO V2 — Direct Text QR

The existing application was inspected before modification:

- systemd service: `pinta-go.service`
- application: `/opt/pinta-go/app.py`
- Gunicorn: 2 workers, `127.0.0.1:5000`
- SQLite: `/opt/pinta-go/pinta.db`
- QR engine: `qrencode 4.1.1`
- Flask: `3.1.3`
- observed service memory before V2: about 46.7 MB
- no new Python dependency was required

Rollback point created:

```text
/opt/pinta-go/app.py.bak-before-v2
```

The backup was SHA-256 checked against the original before work continued.

Development was isolated in:

```text
/opt/pinta-go/app-v2.py
```

The test Gunicorn used one worker on port 5001. It was first bound to `127.0.0.1:5001`, then temporarily bound only to the PINTA Tailscale address so the UI could be tested from JETEK.

### New V2 capability

PINTA GO now supports:

1. Existing **Short URL + QR** workflow.
2. New **Direct Text QR** workflow at `/text`.

Direct Text QR deliberately does **not** write the user's text into SQLite. The text is passed to `qrencode`, a PNG is produced, and the QR was verified by scanning it back with a phone.

Acceptance test passed:

```text
Text input
 -> POST /text/create
 -> qrencode
 -> PNG
 -> phone scan
 -> original Thai text + emoji recovered correctly
```

The existing Short URL workflow was also retested on V2 and passed. This gave a two-part acceptance result: **Short URL passed + Text QR passed**.

After testing, the database was backed up, `app-v2.py` was copied into production `app.py`, Python syntax was checked, `pinta-go.service` was restarted, and the service returned active/running with Nginx still responding HTTP 200.

## Security design before wider public use

The public input must be treated as untrusted.

For Short URL creation, planned hardening is:

```text
Input
 -> scheme/length validation
 -> public destination check
 -> SSRF guard
 -> Turnstile
 -> rate limit
 -> parameterized SQLite
 -> generate
 -> minimal security log
```

Existing SQLite statements already use parameterized queries and the slug is restricted to `a-z 0-9 _ -`.

### URL health checking

A future Short URL check may verify whether a destination is reachable, but it must not become an internal-network scanner. Block localhost, loopback, private/internal addresses, Tailscale/internal destinations, and other non-public targets before any server-side request.

The result should mean **Reachable / Unreachable / Unable to verify**, not “Safe”, because HTTP reachability does not prove that a destination is trustworthy.

Do not fetch arbitrary destination pages for previews/metadata unless genuinely required; unnecessary server-side fetching creates SSRF exposure.

## Minimal anti-abuse plan

Keep it small for the old PINTA hardware:

- Cloudflare Turnstile at generation time, server-side verified.
- Lightweight rate limiting for generation endpoints.
- A machine-wide ceiling/circuit breaker if later needed.
- Minimal rotating security logs.

Initial rate values discussed are only starting points, not production truth: approximately 5 generations/minute/IP and 20/hour/IP, then tune from real traffic. Shared NAT means an IP is not equivalent to one human.

## Privacy / analytics principle

**Public service does not mean surveillance.**

Useful long-term analytics can remain coarse:

- timestamp
- country code / coarse origin
- QR type
- success / blocked / invalid

Full source IP, if retained for abuse/security investigation, should be treated separately as short-lived security data with restricted access and defined retention. Do not log passwords, cookies, Turnstile tokens, or all request headers.

Direct QR content such as text, phone/contact or Wi-Fi data should not be persisted merely for analytics.

If asking what a user created the QR for, make the purpose field optional (for example work / education / personal / other / not specified) rather than inferring it.

## Search-engine principle

When a permanent public domain exists, index the PINTA GO service/help pages if desired, but do not intentionally index individual customer short-link paths. Quick Tunnel hostnames are temporary and are not the place to invest in SEO.

## Scope discipline

Do not turn the Celeron PINTA into a data center.

Current scope:

```text
Short URL + QR
Direct Text QR
(next) Rate Limit
(next) Minimal Log
```

Potential QR types such as phone/contact and Wi-Fi are possible because QR can encode arbitrary structured text, but they are not a reason to expand everything at once.

**Rule:** working service first -> one small capability -> isolated test -> acceptance test -> backup -> production -> stop.

## Field lessons

- Inspect the running service and real source before editing.
- Create and verify a rollback point before changes.
- Test on a separate port instead of experimenting on production.
- Bind test services narrowly; use Tailscale for private UI testing.
- Syntax-check before every restart.
- Retest both the new feature and the old critical workflow.
- No dependency is better than a new dependency when the installed tool already does the job.
- Documentation is part of resilience: if old PINTA dies, rebuild from the blueprint instead of making one old machine irreplaceable.

## Next session

Do not add more QR types yet. The next small hardening block is:

1. lightweight rate limiting
2. minimal rotating security log

Then reassess load and real use before adding anything else.
