# The Hivez status

A small, dependency-free uptime monitor and public status page for [The Hivez](https://thehivez.com/) (the Remote Desk product, app at `https://app.thehivez.com`).

- **Monitor:** a Node 24 script (`scripts/check.mjs`, no npm packages) run every 5 minutes by GitHub Actions.
- **Status page:** a single static file (`docs/index.html`) served by GitHub Pages at `https://status.thehivez.com`.
- **Alerts:** GitHub issues, opened and closed automatically, with an optional Slack/Discord webhook.
- **Storage:** plain JSON committed to this repo (`data/`). No database, no servers, no secrets.

This repo is public on purpose: Actions minutes and Pages are free for public repos, and it only ever contains public URLs.

## What is monitored

Defined in [`monitors.json`](monitors.json). Every monitor is an HTTP `GET`.

| id | Group / name | URL | Healthy means |
|----|--------------|-----|---------------|
| `app` | App / Web app | `https://app.thehivez.com/` | HTTP 200 |
| `api` | App / API | `https://app.thehivez.com/api/health` | HTTP 200 with JSON `status` of `ok` (today it returns a bare `{"status":"ok"}`) |
| `livekit` | Calls / Voice & video (LiveKit signaling) | `https://app.thehivez.com/rtc/validate` | HTTP **401** with a body containing `no permissions` |
| `website` | Website | `https://thehivez.com/` | HTTP 200 |

About the `livekit` monitor: an unauthenticated request to LiveKit's `/rtc/validate` is rejected with
`401 no permissions to access the room` when the SFU is alive, so a 401 is the *healthy* answer. If LiveKit is down the
reverse proxy answers 502/504 instead, which is not in `expect.status`, so the monitor goes down. It proves the signaling
endpoint is reachable through the proxy, not that media (UDP 7882 / TCP 7881) flows.

A monitor is **down** on timeout, network error, wrong HTTP status, missing body text or an unexpected JSON status.
Before declaring down, a failing check is retried twice, 10 seconds apart, and the best of the three attempts wins, so a
single dropped request does not flap. TLS certificate expiry is checked for every HTTPS hostname.

### Planned: `api-deep` (do not add yet)

A fifth monitor is planned for the server's deep health endpoint. **It is not in `monitors.json` on purpose.**

```json
{
  "id": "api-deep",
  "name": "API dependencies",
  "group": "App",
  "url": "https://app.thehivez.com/api/health/deep",
  "expect": { "status": [200], "jsonStatusIn": ["ok", "degraded"] },
  "degradedJsonStatusIn": ["degraded"],
  "timeoutMs": 20000
}
```

The response is `{ "status": "ok" | "degraded" | "unhealthy", "checks": { "database": ..., "redis": ..., "livekit": ... } }`
and the endpoint returns 503 only when the database is down. `degraded` (for example Redis or LiveKit failing) shows as an
amber "Degraded" state with the failing check names in the incident, and `unhealthy` / 503 is Down.

That endpoint is being added to the server separately. **Only add this entry to `monitors.json` after the server change is
deployed to production.** Before that the URL returns a 404, the first run would see a failing monitor and open a false
"down" incident (and fire the webhook).

## How alerts work

For each monitor the workflow keeps at most one open issue per kind, labelled `incident`, `monitor:<id>` and `down` or `degraded`:

- Goes down or degraded: an issue is opened ("🔴 Web app is down"; a degraded issue reads "🟠 <name> is degraded") with the URL, time in UTC, HTTP code,
  error, response time and a link to the workflow run.
- Still down: a comment after the first hour, then every 4 hours, then once a day (measured from when the incident opened).
- Flapping: if a monitor goes down or degraded again within 30 minutes of its incident closing, that issue is **reopened** with a
  comment instead of opening a new one. Only issues the monitor closed itself are reopened (the last comment is its "Recovered after"
  comment and no person closed it); an issue a human closed is left alone and a new one is opened. The webhook always posts the alert
  after a recovery notice, and only throttles a repeated alert within 30 minutes.
- Degraded alerts only appear once the planned `api-deep` monitor (below) is added: none of the current four monitors sets `degradedJsonStatusIn`, so today a monitor is either up or down.
- Recovers: a "Recovered after ..." comment and the issue is closed.
- TLS certificate under 14 days: one `tls` issue per host ("🔒 TLS certificate for host expires in N days"), closed once renewed.

GitHub notifies people who are watching the repo about new issues, so to get alerted:

1. **Watch** the repo, choose *Custom* and tick **Issues**.
2. Install the **GitHub mobile app** and enable push notifications for a push on your phone.

Optional chat alerts: add a repository secret `ALERT_WEBHOOK_URL` (Settings, Secrets and variables, Actions) holding a Slack or
Discord webhook URL. On every incident open and recover the script POSTs `{"text": "...", "content": "..."}` (both fields, so
the same payload works for Slack and Discord). Webhook failures never fail the run.

Text that comes from the monitored service (error messages, health-check details) is truncated to 300 characters and shown in
a code block, so it cannot @mention anyone; webhook payloads set `allowed_mentions: {parse: []}` and escape `<`, `>` and `&`.
All GitHub and webhook calls share a 150 second budget so a slow API cannot stall the job.

The script always exits 0, and a push that cannot be completed (a conflict, or a second rejected push) is logged and that
sample is dropped, so the workflow does not go red every 5 minutes. Only genuine infrastructure failures (checkout, setup-node)
can turn it red. Alerting is entirely through issues and the webhook.

## How data is stored (and its granularity)

Every run starts from the last **committed** state, so a sample that is not committed is lost. To avoid 288 commits a day,
a run commits only when:

- any monitor changed state, or an incident issue was opened, commented or closed, or
- it is the hourly checkpoint: (UTC minute < 5 AND the last commit is at least 30 minutes old) OR the last commit is at least 57 minutes old, or
- `data/status.json` has no `generatedAt` yet (the initial empty state shipped in this repo).

`check.mjs` signals this by writing the git-ignored flag file `data/.commit`; the workflow commits `data/` and
`docs/status.json` as `github-actions[bot]` with the message `status: update` (one `git pull --rebase -X theirs` retry if the push is rejected; on a conflict the sample is dropped).
The commit deliberately has no `[skip ci]`, so the GitHub Pages build still runs for it.
The job checks out `main` explicitly, so a run that was queued for a while still starts from the branch tip.

Consequences, worth knowing:

- History (`data/history/<id>.json`) has **roughly hourly granularity plus every state change**, kept for 90 days.
- A short outage is always captured (the down transition and the recovery are both committed), but the
  "last checked" time on the page can be up to an hour old while everything is stable.

`data/status.json` is the state file; `docs/status.json` is an identical copy because GitHub Pages only serves `docs/`.
The page fetches the copy with a cache-busting query string and refreshes every 60 seconds. The shipped file is an empty
shape (`"generatedAt": null`, no monitors), which the page renders as "Waiting for the first check".

## Uptime math (all UTC)

- Uptime percentages are weighted by the time between samples: a sample's state holds until the next sample, but never longer
  than 3 hours (a longer gap counts as "no data", not as up or down). Degraded counts as up. Windows are 24 hours, 7 days
  and 90 days.
- Daily bars use the plain **UTC calendar day** (midnight to midnight UTC, always 24 hours, no daylight-saving handling):
  green at 99.5% or above, amber at 95% or above, red below, grey with no data.
- All times on the page, in issues and in webhook messages are UTC.

## Setup (once, by hand)

Nothing here has been done yet: this directory is a local-only git repo with a single commit.

- [ ] Create the **public** GitHub repo `Shining-Technologies/the-hivez-status` (public is needed for free Pages and Actions).
- [ ] Add it as `origin` and push `main` (`git remote add origin git@github.com:Shining-Technologies/the-hivez-status.git && git push -u origin main`).
- [ ] Settings, Pages: **Deploy from a branch**, branch `main`, folder `/docs`. Set the custom domain to `status.thehivez.com`
      (`docs/CNAME` already contains it).
- [ ] DNS: add a `CNAME` record `status` pointing to `shining-technologies.github.io`.
- [ ] Once the certificate is issued, tick **Enforce HTTPS** in Pages.
- [ ] Settings, Actions, General, Workflow permissions: **Read and write permissions** (the workflow also declares
      `contents: write` and `issues: write`, but an org/repo default of read-only can override it). Enable workflows in the Actions tab.
- [ ] Run **check** once with *Run workflow* to confirm it, then it runs on its own every 5 minutes.
- [ ] Watch the repo (Custom, Issues) and install the GitHub mobile app for push alerts.
- [ ] Verify `thehivez.com` as a GitHub Pages domain at the organisation level (Shining-Technologies → Settings → Pages → Add a domain, via a DNS TXT record). This prevents anyone else claiming `status.thehivez.com` (subdomain takeover) if the Pages site is ever removed.
- [ ] Optional: add the `ALERT_WEBHOOK_URL` repository secret (Slack or Discord webhook).
- [ ] **Check presence-space's IP blocker / rate limiter** (`/api/security/*` IP allow/deny, request rate limits) so it does not
      ban GitHub Actions runner IPs. The monitor hits `/api/health`, `/`, and `/rtc/validate` from shared Azure IPs every
      5 minutes (up to 3 requests each when failing); `/rtc/validate` returns 401 by design, so make sure repeated 401s from one IP
      are not counted as failed-auth abuse. Consider allowlisting `the-hivez-status-monitor` (the User-Agent) or exempting
      those paths if anything is auto-blocked.

Notes:

- GitHub's cron can be delayed by several minutes under load, so "every 5 minutes" is best effort.
- Scheduled workflows in public repos are **disabled after 60 days without repo activity**. The hourly bot data commits count as
  activity, so the monitor keeps itself alive; if it ever gets disabled anyway, re-enable it in the Actions tab.

## Adding a monitor

Append an entry to `monitors.json` and push:

```json
{
  "id": "docs",
  "name": "Help centre",
  "url": "https://help.thehivez.com/",
  "expect": { "status": [200], "bodyIncludes": "Hivez" },
  "timeoutMs": 15000
}
```

- `id`: lowercase slug, unique; used for the history file and the `monitor:<id>` label.
- `expect.status`: accepted HTTP codes (may be a non-200 code, see `livekit`). `expect.bodyIncludes`: text that must appear. `expect.jsonStatusIn`: accepted values of the JSON `status` field.
- `degradedJsonStatusIn`: which of those JSON statuses mean "degraded" rather than "up".
- `group` (optional): monitors sharing a group are shown in one card on the page.
- `timeoutMs` defaults to 15000.

Only public URLs belong here; this repo is public.

## Running locally

```bash
npm run check:dry                   # probe everything and print the table; writes no files, opens no issues, sends no webhook
npm run check                       # real run: writes data/ and docs/status.json (and, with a token, touches issues)
npm test                            # unit tests for the pure functions (node:test)
python3 -m http.server -d docs      # preview the page
```

Without `GITHUB_TOKEN` and `GITHUB_REPOSITORY` no issues or webhooks are touched; the script logs what it would have done.
`RETRY_DELAY_MS` shortens the retry delay for testing. Requires Node 24. (On Node 22 and later `node --test scripts/` is read as a
glob rather than a directory, so the npm script passes the test file explicitly.) Avoid `npm run check` locally unless you mean to
overwrite the committed state files.

## Layout

```
monitors.json                 what to check
scripts/check.mjs             the monitor (and its exported pure functions)
scripts/check.test.mjs        node:test unit tests
.github/workflows/check.yml   5-minute schedule, commits data
data/status.json              current state (committed; starts empty)
data/history/<id>.json        90-day sample history (committed; starts empty)
docs/index.html               the status page
docs/status.json              copy of data/status.json for Pages
docs/CNAME, docs/.nojekyll    custom domain, disable Jekyll
```

Ported from the Vistra status monitor with these differences: UTC instead of Australia/Sydney, Hivez branding and palette,
a plain UTC day-window implementation instead of the DST-aware one, and a fresh empty state.
