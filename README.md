# brain fog

**Crypto phishing-domain & brand-impersonation early warning system (open source, read-only, public data only).**

Register your brand, and Brain Fog watches Certificate Transparency logs for lookalike domains, correlates them with public
scam feeds, scores them with a transparent model, preserves timestamped evidence, alerts your team on Discord/Telegram, and
drafts takedown reports. This repository is the **v0.1 backend MVP** (FastAPI + SQLite).

> Automation never declares a site "confirmed phishing". It produces `unverified` or `suspected` findings with reasons;
> a human reviews and confirms. Expect false positives and tune thresholds.

## Quick start (offline demo, no network)

```bash
python -m venv .venv && source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt
python -m app.cli demo                                   # seeds a fictional brand, replays sample CT + feed data
python -m app.cli report 5                               # takedown draft for the top alert (id may differ)
pytest                                                   # 50+ offline tests
```

## Run the API

```bash
playwright install chromium                              # needed only for screenshots
cp .env.example .env
python -c "import secrets; print(secrets.token_urlsafe(32))"   # paste into BRANDHOOK_API_KEY in .env
uvicorn app.main:app --host 127.0.0.1 --port 8000        # docs at http://127.0.0.1:8000/docs (development mode)
```

```bash
KEY=your-api-key
curl -s -X POST localhost:8000/brands -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
     -d "$(python -c "import json;print(json.dumps(json.load(open('sample_data/brands.sample.json'))[0]))")"
curl -s -X POST localhost:8000/domains/check -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
     -d '{"brand_id":1,"domain":"exampleswap.io.verify-wallet.click"}'          # dry run: score + reasons
curl -s -X POST localhost:8000/admin/feeds/refresh -H "X-API-Key: $KEY"        # download public scam feeds
curl -s -X POST "localhost:8000/admin/ct/run?brand_id=1" -H "X-API-Key: $KEY"  # run CT monitor now
curl -s "localhost:8000/alerts?min_level=high" -H "X-API-Key: $KEY"
curl -s -X PATCH localhost:8000/alerts/1/verdict -H "X-API-Key: $KEY" -H "Content-Type: application/json" \
     -d '{"verdict":"confirmed","reviewer":"alice","note":"checked page and DNS"}'
curl -s -X POST localhost:8000/alerts/1/takedown-report -H "X-API-Key: $KEY" -H "Content-Type: application/json" -d '{"target":"registrar"}'
```

With `BRANDHOOK_SCHEDULER_ENABLED=true` (default) the server polls crt.sh every 30 minutes and refreshes feeds every 6 hours.
Set `BRANDHOOK_DISCORD_WEBHOOK_URL` and/or `BRANDHOOK_TELEGRAM_BOT_TOKEN` + `BRANDHOOK_TELEGRAM_CHAT_ID` to receive alerts.
All settings are documented in `.env.example`.

## How it works

```
crt.sh search terms (brand label, keywords, a few typo/homoglyph variants)
        -> candidate domains -> skip official/allowlisted/opted-out
        -> name analysis (RapidFuzz edit distance, homoglyph skeletons, combosquatting, embedded official domain, scam terms)
        -> public scam feed lookup (MetaMask eth-phishing-detect, ScamSniffer; parent-domain match)
        -> DNS + RDAP enrichment (domain age, registrar, abuse contact; registrant data is discarded)
        -> risk score + reasons -> alert (medium+) -> evidence (high+) -> notification (high+)
```

| Level | Score | Stored | Screenshot/HTML | Notified |
|-------|-------|--------|-----------------|----------|
| low | 0-24 | no | no | no |
| medium | 25-49 | yes | no | no |
| high | 50-74 | yes | yes | yes |
| critical | 75-100 | yes | yes | yes |

Every weight is in `app/detection/scoring.py::WEIGHTS`; thresholds are env vars. Evidence for each alert is stored under
`data/evidence/alert-<id>/<UTC timestamp>/` with SHA-256 hashes in `manifest.json` (DNS, RDAP, certificate, feed matches,
screenshot, HTML snapshot, detection record).

## API summary (all except `/health` need `X-API-Key`)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/health` | liveness |
| POST/GET/DELETE | `/brands`, `/brands/{id}` | register / list / deactivate protected brands |
| POST | `/domains/check` | score one domain (dry run) or persist it |
| GET | `/alerts`, `/alerts/{id}` | list (filters: brand_id, verdict, min_level) / detail with reasons + evidence |
| PATCH | `/alerts/{id}/verdict` | human review (`confirmed` only possible here) |
| POST | `/alerts/{id}/capture-evidence` | fresh screenshot + HTML snapshot |
| GET | `/alerts/{id}/evidence/{eid}/file` | download evidence (HTML served as text/plain) |
| POST/GET | `/alerts/{id}/takedown-report(s)` | draft takedown (requires suspected/confirmed) |
| POST | `/admin/ct/run`, `/admin/feeds/refresh` | run jobs now |
| GET | `/admin/feeds/status` | feed freshness |
| GET/POST/DELETE | `/admin/optouts` | operator-managed opt-out list |

## Safety design

Public data only, passive browsing only (one page load; no clicks, typing, wallet injection or downloads), SSRF guard on every
fetch, honest User-Agent with your contact, polite rate limits for crt.sh/RDAP, defanged alerts, API key auth + rate limit,
audit log. See `SECURITY.md`. Run evidence capture in a container with no route to internal networks.

## Known limitations (MVP)

- crt.sh search is substring-based, so typos are found only through the few generated variants (`BRANDHOOK_CT_MAX_VARIANT_QUERIES`).
  For full-stream fuzzy matching add a CertStream-style provider (`CTProvider` protocol in `app/detection/ct.py`).
- crt.sh is occasionally slow or returns errors; failures are logged and retried on the next run.
- One alert per hostname (a leading `www.` is collapsed); grouping by registrable domain and infrastructure clustering are planned.
- Name heuristics are not perfect: expect false positives on short brand names. Use `allowlist_domains` and `PATCH .../verdict`.
- Feeds are third-party data under their own licences; PhishFort/other feeds can be added in `app/detection/feeds.py`.

## Roadmap

v0.2: email notifier, Cert Spotter provider, favicon + template similarity, re-registration watch, Next.js dashboard (server-side API proxy),
Docker image. v1.0: infrastructure clustering, social/GitHub/npm impersonation monitors, multi-tenant auth, Postgres.

## License

MIT
