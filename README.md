# TRMNL Home Dashboard

Combines the date + today's high/low, current weather + condition icon, a next-6-hours strip, a
5-day forecast strip, sunrise/sunset, and a sports panel showing the next game
for whichever teams you list in [`teams.json`](teams.json) — soonest first,
with out-of-season teams dropped — into a single TRMNL screen, using a
**private plugin (webhook strategy)**: a script fetches all the data and
pushes it to TRMNL on a schedule via GitHub Actions; TRMNL renders it through
a template you control.

## How it fits together

- [`fetch_and_push.py`](fetch_and_push.py) — pulls weather/sun from Open-Meteo
  and each configured team's next game or racing series' next race from either
  football-data.org or ESPN's public API (whichever that entry's `provider`
  says). `collect_games()` merges them into one `games` list, sorted by
  kickoff, with any entry whose next event is past the horizon (see
  `SPORTS_HORIZON_DAYS`) removed — and any entry whose fetch fails
  (bad abbreviation, API outage, etc.) logged as a warning and skipped,
  rather than blocking everyone else's games from being pushed. It all goes
  to your TRMNL webhook URL as `merge_variables`.
- [`teams.json`](teams.json) — the list of teams to track. Edit this file to
  add/remove/swap teams; no Python required for the common case. See
  [`SETUP.md`](SETUP.md) for the field reference (ESPN sport paths, racing
  series, team abbreviations, football-data competition codes). Tracking a
  team via a provider that isn't built in (say, a cricket API) needs a
  small Python addition, not just a `teams.json` edit — see SETUP.md's
  "Using a provider that isn't built in yet" for what that involves, and
  [`Next Step Ideas.md`](Next%20Step%20Ideas.md) idea #2 for making that
  config-only in a future version.
- [`template.html`](template.html) — the markup you paste into TRMNL's private
  plugin editor; it interpolates the pushed variables.
- [`.github/workflows/update-dashboard.yml`](.github/workflows/update-dashboard.yml) —
  runs the script on GitHub Actions. It has both a `schedule:` (cron) trigger
  and a `workflow_dispatch` trigger. GitHub's cron is unreliable for new /
  low-traffic private repos (see step 7 below), so the dependable way to keep it
  ticking is an external timer hitting `workflow_dispatch`.
- [`preview.py`](preview.py) — renders `template.html` with TRMNL's real
  framework CSS into a local `preview.html` so you can check the layout in a
  browser before the device is set up.

## Previewing the layout without a device

```
python preview.py          # live data if your .env has creds, else sample data
python preview.py --fake    # sample data, no network calls
python preview.py --no-open # write preview.html without opening a browser
```

It loads TRMNL's own `plugins.css` from `trmnl.com`, so spacing and sizing
match the device (just in colour rather than 1-bit black & white). Handy for
deciding whether you've got room for more plugins on the screen.

## One-time setup

> New to this? [`SETUP.md`](SETUP.md) is a step-by-step guide for a first-time
> coder on Windows — installing Git, Python, and VS Code, cloning your own copy,
> and getting a preview running before you touch any of the steps below.

### 1. Create the TRMNL private plugin

1. In the TRMNL dashboard: **Plugins → Private Plugin → Add New**.
2. Strategy: **Webhook**.
3. Save it — TRMNL gives you a **Plugin Settings UUID**. Your webhook URL is:
   `https://usetrmnl.com/api/custom_plugins/<UUID>`
4. Open the plugin's **Markup** editor, switch the view to **Full**, and paste
   in the contents of [`template.html`](template.html). TRMNL's live previewer
   lets you tweak spacing/sizing for the actual OG display — treat the
   provided layout as a first draft.
5. Add the plugin to a playlist (or make it the only plugin) so it's what
   shows on the device.

### 2. Get a football-data.org API key

Only needed if [`teams.json`](teams.json) has a `"provider": "football-data"`
entry (ESPN-only setups can skip this). Free tier, no credit card:
<https://www.football-data.org/client/register>. The free tier covers most
of Europe's top leagues — see [`SETUP.md`](SETUP.md) for competition codes.

### 3. Decide your weather location

Grab lat/lon for wherever you want weather/sunrise-sunset for (not
necessarily Liverpool) — e.g. from Google Maps ("what's my lat/lon here").

### 4. Push this to GitHub

```
git add -A
git commit -m "Initial TRMNL dashboard project"
gh repo create trmnl-dashboard --private --source=. --push
```

(Or create the repo on GitHub first and `git remote add origin ...` yourself.)

### 5. Configure GitHub Actions secrets and variables

In the repo: **Settings → Secrets and variables → Actions**.

**Secrets** (sensitive):
- `TRMNL_WEBHOOK_URL`
- `FOOTBALL_DATA_API_KEY`

**Variables** (not sensitive):
- `WEATHER_LAT`, `WEATHER_LON`
- `TIMEZONE` — IANA name, e.g. `Europe/London` or `America/New_York`
- `TEMP_UNIT` — `celsius` or `fahrenheit`
- `SPORTS_HORIZON_DAYS` — optional; hide a team whose next game is more than
  this many days away. Leave it unset to use the default of 30.

### 6. Test it

- Locally (PowerShell): copy `.env.example` to `.env`, fill in your values,
  then install and run. The script auto-loads `.env` via `python-dotenv`
  (`.env` is gitignored — never commit it):
  ```powershell
  pip install -r requirements.txt
  copy .env.example .env   # then edit .env
  python fetch_and_push.py
  ```
  If you'd rather not use a `.env` file, set the same names as `$env:`
  session variables instead — the script reads plain environment variables
  too, and real env vars win over `.env`.
- On GitHub: **Actions → Update TRMNL dashboard → Run workflow** to trigger it
  manually and check the logs.
- Force TRMNL to pull the latest state via the plugin's "Force Refresh" in
  its dashboard if you don't want to wait for the device's normal refresh
  cycle.

### 7. Make the schedule reliable (external trigger)

GitHub's `schedule:` (cron) trigger is best-effort: under load GitHub silently
drops scheduled runs, and brand-new or low-traffic private repos are
deprioritised heavily — a fresh repo can go hours or days before cron fires at
all. The workflow also exposes a `workflow_dispatch` trigger, so the fix is to
have a reliable external service call that on a timer. Everything stays in this
repo; the external service only presses "Run workflow".

**a. Create a fine-grained personal access token**

GitHub → **Settings → Developer settings → Personal access tokens → Fine-grained
tokens → Generate new token**:

- **Name:** `trmnl-dashboard-cron`
- **Resource owner:** your account
- **Expiration:** 1 year — set a calendar reminder to regenerate it; the job
  stops working silently when it lapses
- **Repository access:** *Only select repositories* → `trmnl-dashboard`
- **Permissions → Repository permissions → Actions:** *Read and write*
  (Metadata read-only is added automatically)
- Generate, and copy the `github_pat_…` string.

That token can *only* trigger workflows in this one repo — it can't read your
code or push commits.

**b. Check it works from your machine**

```powershell
curl.exe -i -X POST `
  -H "Accept: application/vnd.github+json" `
  -H "Authorization: Bearer github_pat_XXXX" `
  -H "X-GitHub-Api-Version: 2022-11-28" `
  https://api.github.com/repos/<your-username>/trmnl-dashboard/actions/workflows/update-dashboard.yml/dispatches `
  -d '{\"ref\":\"main\"}'
```

Expect `HTTP/2 204` and an empty body. A new run should appear under
**Actions → Update TRMNL dashboard** within a few seconds.

**c. Schedule it on cron-job.org**

Sign up free at <https://console.cron-job.org>, then **Create cronjob**:

| Field | Value |
| --- | --- |
| Title | TRMNL dashboard refresh |
| URL | `https://api.github.com/repos/<your-username>/trmnl-dashboard/actions/workflows/update-dashboard.yml/dispatches` |
| Schedule | custom: minutes `13,43`, every hour, every day |

Under **Advanced / "Show more"**:

- **Request method:** `POST`
- **Request body:** `{"ref":"main"}`
- **Headers:**
  - `Accept: application/vnd.github+json`
  - `Authorization: Bearer github_pat_XXXX`
  - `X-GitHub-Api-Version: 2022-11-28`
  - `Content-Type: application/json`

Save, and enable **"Notify me by email if the job fails"** so you hear about it
when the token expires. cron-job.org counts the `204` as success; if it flags a
non-2xx expectation, set the expected status to `204`.

The trade-off: your token sits on cron-job.org's servers. For this project the
worst case is "someone can re-run your dashboard workflow", so it's a fine risk.
To keep the secret in your own infrastructure instead, a Cloudflare Worker with
a Cron Trigger does the same job.

You can leave the `schedule:` block in the workflow as a fallback — if GitHub's
cron ever does fire you just get a harmless duplicate push, still well under
TRMNL's 12-pushes/hour cap. Remove it only if you want a single source of truth.

## Notes / known limitations

- **GitHub scheduled runs are unreliable**: the `schedule:` cron in the workflow
  is best-effort and GitHub drops runs under load, worst of all for new /
  low-activity private repos. Step 7 above (external trigger via
  `workflow_dispatch`) is what actually keeps the dashboard current; the cron is
  just a fallback.

- **Offseasons**: a team with no scheduled game, or whose next game is more
  than `SPORTS_HORIZON_DAYS` (default 30) away, is dropped from the panel
  entirely rather than showing a fixture months out — so once a team's
  season ends, it disappears until its next game is within the horizon. If
  every team is out of season the panel shows a "No games in the next N days"
  line. `get_espn_next_game()` also filters to games whose status is `pre` and
  whose start time is still in the future, so a *postponed* game (which ESPN
  leaves looking not-completed) can't show up as the "next" one. For F1,
  `get_espn_next_race()` selects the scheduled `Race` session, not practice
  or qualifying.
- **Bad team config degrades gracefully**: if a `teams.json` entry has a typo
  (wrong ESPN abbreviation, a `match` string that doesn't exist in that
  competition, a transient API error) `collect_games()` logs a `WARNING:` to
  stderr for that entry and drops it, rather than failing the whole run — the
  remaining teams still push. Only a structurally broken `teams.json` itself
  (missing file, invalid JSON, missing required fields) is fatal.
- **Rate limits**: football-data.org free tier is 10 requests/minute; each
  `"provider": "football-data"` entry in `teams.json` costs 2 calls per run
  (list the competition's teams, then that team's matches) — a few such
  entries are fine, but many will approach the per-minute cap faster.
  `football_data_get()` reads the `X-Requests-Available-Minute` /
  `X-RequestCounter-Reset` response headers and backs off on a 429, per
  football-data.org's guidance. ESPN entries hit an unauthenticated public
  endpoint with no documented limit. TRMNL webhook free tier is 12
  pushes/hour — the 30-minute schedule uses well under that.
- **Payload size**: TRMNL free tier caps webhook payloads at 2KB. The default
  5-team payload (current conditions + 6 hourly + 5-day forecast + five
  teams) is ~1.4KB — still under, but the margin is thinner now, so watch it
  as you add teams or fields. The weather-icon SVGs live in the template, not
  the payload, so they don't count.
- **Sports panel height**: five teams in season at once fills the right
  column. It's `overflow: hidden`, so adding more teams (or very long
  opponent names) would clip the bottom rather than reflow — trim
  `teams.json` or drop to a one-line-per-team format if you hit that.
- **Weather icons**: the script sends a keyword (`clear`, `partly`, `cloudy`,
  `fog`, `rain`, `snow`, `storm`) for current conditions and for each hourly /
  daily slot. `template.html` has one `.wx--<keyword>` CSS rule per keyword,
  each a URL-encoded line-icon SVG used as a `mask` and painted in
  `currentColor`. To add a keyword: map it in `WMO_ICONS` (Python) and add one
  `.wx--<keyword>` rule (encode the SVG the same way as the others).
