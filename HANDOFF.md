# Handoff — U.S. Macro Dashboard (production version)

A single-page macro dashboard (`index.html`) that a GitHub Actions pipeline refreshes weekly from FRED / BLS / EIA,
annotates with Claude-written commentary, validates, and publishes to GitHub Pages. This repo is the **production
version only**; it was exported from the author's `main` branch with no git history.

For architecture, agent roles and the operational gotchas learned the hard way, read `README.md`, `CLAUDE.md`
(the "Known Gotchas" section is the institutional memory) and `METHODOLOGY.md`.

## One-time setup

1. **Create the repo** on your git host and push this tree to its default branch (`main`). The workflows assume `main`.
2. **Pages:** Settings → Pages → **Source: GitHub Actions**. (The site is deployed by `actions/deploy-pages`; a plain
   `git push` does *not* update the live site — only a run of the `Weekly Macro Update` workflow does.)
3. **Actions permissions:** Settings → Actions → General → Workflow permissions → *Read and write* (the pipeline commits
   refreshed data back to `main`).
4. **Secrets** (Settings → Secrets and variables → Actions):

   | Secret | Needed for | Required? |
   |---|---|---|
   | `FRED_API_KEY` | data collection (most series) | **yes** |
   | `BLS_API_KEY` | jobs / CPI / wages | **yes** |
   | `EIA_API_KEY` | oil prices | **yes** |
   | `ANTHROPIC_API_KEY` | AI commentary (Agent 3), earnings extraction (Agent 9), vision review (Agent 8) | yes for full function — see "AI steps fail soft" below |
   | `RESEND_API_KEY`, `EMAIL_FROM`, `EMAIL_TO` | emailing the weekly briefing | optional — **leave unset to run with no email** (the publisher logs "skipping email") |

   Optional: set the `DASHBOARD_URL` env var if the briefing email should link somewhere other than the derived
   `https://<owner>.github.io/<repo>/` (e.g. GitHub Enterprise Pages).
5. **First run:** Actions → *Weekly Macro Update* → Run workflow. Watch the log; then open the Pages URL.
   Consider running with the email secrets unset first.

## What runs when

| Workflow | Trigger | Purpose |
|---|---|---|
| `briefing.yml` — Weekly Macro Update | Sat 12:00 UTC + manual | Full pipeline: collect → analyze → AI commentary → render → validate → repair-observe → snapshot → archive → commit → **deploy** → healthcheck → email |
| `earnings_agent.yml` — Agent 9 | days 10-28 of Jan/Apr/Jul/Oct, 22:00 UTC + manual | Extracts bank earnings-call commentary; the verbatim gate must pass before it commits |
| `smoke-tests.yml` | PR / push to `main` | Offline tests + visual QA |
| `api-contract.yml` | Tue 10:00 UTC | Checks FRED/BLS/EIA response shapes still parse |
| `backtest.yml` | manual | Re-runs the Oil Impact Chain threshold calibration |

## Recurring human work (not automated)

- **Every earnings season:** update `data/earnings_calendar.json` for the new quarter (`scripts/rollover_calendar.py`
  drafts it), then **verify every transcript URL**. Only `fool.com` URLs have worked from GitHub runners —
  benzinga / seekingalpha / investing.com return HTTP 403. The cron window closes on day 28; `workflow_dispatch`
  works any time.
- **Also every quarter:** the Banks tab contains hand-written blocks that the pipeline does not touch (see below).

## Known issues at handoff

1. **Banks tab is mixed-quarter.** The per-bank commentary cards are fed by Agent 9 and are current (Q3 2026). The
   comparison table (`BANK_RESULTS`), the summary tiles derived from it, the NII-guidance and NCO charts,
   `BANK_THEMES` ("Cross-Bank Themes") and the panel titles / source footers are hand-written **Q1 2026** content in
   `index.html`. Refreshing them needs sourced Q3 figures and verbatim quotes (see the factuality rule in `CLAUDE.md`).
2. **Fed Rates tab — Wall St forecast table is stale.** The GS/JPM/MS 2026 targets date from Jan 2026 (flagged with a ⚠
   badge). There is no live analyst-forecast feed; a human must update them.
3. **AI steps fail soft.** If `ANTHROPIC_API_KEY` is missing or out of credit the workflow still reports *success*
   (regex fallbacks keep a few sentences numerically current) but no fresh AI commentary or earnings extraction
   happens. Check run logs for `BadRequestError` / `credit balance`; see the *Resolved incident* in `CLAUDE.md`.
4. `tests/test_smoke.py`: run it before pushing (`python tests/test_smoke.py`).

## What was deliberately left out of this repo

The author's parallel `dev` branch and its tooling (`briefing-dev.yml`, prod-vs-dev comparison job and scripts, the
local preview script), the Claude Code project config (`.claude/`), design mockups and one-off preview pages, an
unrelated `consumer.html` / `jeopardy.html`, one-off diagnostics (`umich-probe.yml`), generated screenshots and rolling
snapshots (both regenerate on the first run), and all git history. No secrets are committed.

Data included: the current baseline files the pipeline diffs against (`data/*.json`), the permanent
`data/monthly_archive/` history, and the **current-quarter** earnings transcripts (`data/transcripts/Q3_2026/`).
Those transcripts are third-party (Motley Fool) text kept so the verbatim gate can verify quotes — have your
legal/compliance team confirm you are comfortable storing them in the repo.
