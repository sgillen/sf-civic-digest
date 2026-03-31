---
name: sf-civic-digest
description: Track San Francisco city government activity — Board of Supervisors, Land Use committee, Public Safety committee, SFMTA Engineering Public Hearings, SF Planning Commission, Historic Preservation Commission, Zoning Administrator hearings, Planning project notices, and SF.gov district events. Fetches agendas and recaps from SF Legistar, sfmta.com, sfplanning.org, the sf.gov Wagtail CMS API (api.sf.gov), and Socrata open data via curl-based scraping. Filtered by supervisorial district, neighborhood, streets, or topic keywords. Use when a user asks what's happening at City Hall, wants to track legislation or development in their neighborhood, wants a weekly or daily SF civic digest, asks about upcoming public hearings, planning variances, or wants to know what their supervisor is working on. SF-specific.
---

# SF Civic Digest — SKILL.md

> **Cold-start note:** This file is designed to be self-sufficient. An agent reading only SKILL.md should be able to run reports, generate digests, and understand the full data stack. No other context required.

---

## Quick Start

```bash
# Step 1: Find the skill root (wherever this SKILL.md lives)
SKILL_DIR="$(dirname "$(realpath "$0")")"   # if running a shell script
# or just note the directory containing this SKILL.md file

# Step 2: Run scripts using that absolute path (--district N required)
python3 "$SKILL_DIR/scripts/sf_weekly_digest.py" --district 5 --json

# Step 3: Synthesize the JSON output into a narrative report (see STYLE.md)
```

**District:** Pass `--district N` (1–11) to any script. User preferences and neighborhood context live in `USER.md` — read it before writing a report. There is no `civic_config.json` or `civic_config.example.json`; those files have been removed.

**⚠️ Path rule — critical for any agent (any agent, IDE, or human):**
- All scripts must be run with the **absolute path to the scripts/ directory**
- Find it: `scripts/` is always a sibling of this SKILL.md file
- Example: if SKILL.md is at `/home/alice/.openclaw/workspace/skills/sf-civic-digest/SKILL.md`, then scripts are at `/home/alice/.openclaw/workspace/skills/sf-civic-digest/scripts/`
- **Never use bare relative paths like `scripts/foo.py`** — they break when the working directory is anything other than the skill root
- When in doubt: `realpath skills/sf-civic-digest/scripts/sf_civic_digest.py` gives you the absolute path

---

## Architecture

**Scripts are data pipelines. The agent writes the narrative.**

The scripts fetch, parse, dedup, and output JSON. The agent (you) reads that JSON and synthesizes it into a report following the style guide. There is no auto-generated report — the editorial judgment is yours.

```
sf_weekly_digest.py --json
    └── calls all subscripts → returns combined JSON
            │
            ▼
    Agent reads JSON + STYLE.md
            │
            ▼
    Agent writes narrative report
```

For report generation, see **Report Generation** section below.

---

## All Scripts

**Canonical script location:** `skills/sf-civic-digest/scripts/`

| Script | What it does | Method | Notes |
|--------|-------------|--------|-------|
| `sf_weekly_digest.py` | Master aggregator — calls all subscripts | — | Use `--json` for machine-readable output; pass to agent for synthesis |
| `sf_civic_digest.py` | Board of Supervisors + all committees via Legistar | curl | `--daily` for delta only; `--days N`; state in `sf_civic_state.json` |
| `sfmta_hearings.py` | SFMTA Engineering Public Hearings | curl + PDF | Street changes, parking meters, signals. Every ~2 weeks. |
| `sf_planning_notices.py` | SF Planning project notices (Section 311) | curl | Earliest signal layer — before commission hearings. Needs `Accept: text/html` header. |
| `sf_planning_commission.py` | Planning Commission, HPC, ZA full agendas | curl + PDF | Server-rendered HTML (no browser). `--body hpc` for HPC, `--body za` for ZA. |
| `sf_board_of_appeals.py` | Board of Appeals | Wagtail API | sf.gov CMS API (`api.sf.gov`). 1st & 3rd Wednesdays, 5pm. No browser needed. |
| `sf_building_permits.py` | Building permits citywide | Socrata REST | Every permit filed. Classified HIGH/NOTABLE/noise. State-tracked. |
| `sfgov_events.py` | Supervisor office hours, district events | REST/JSON | Very sparse — supervisor events only. Low signal. |
| `sf_311.py` | 311 service requests by district | Socrata REST | Spike detection. Query two 7-day windows for trend comparison. |
| `sf_ethics.py` | Lobbyist contacts + campaign contributions | RSS + Socrata | Contacts: `5f5n-tdbf`. Contributions: `e6py-fg8b`. Column names have NO underscores. |
| `sf_rent_board.py` | Rent Board commission meetings | Wagtail API | sf.gov CMS API (`api.sf.gov`). No browser needed. Rent increase: links to sf.gov (no hardcoded value). |
| `sf_sfmta_board.py` | SFMTA Board of Directors | curl | 1st & 3rd Tuesdays 1pm. Service changes, fare decisions, bike/ped safety. |
| `sf_rec_park.py` | Rec & Park Commission | curl (Granicus) | `view_id=91`. sfrecpark.org is unreachable from server — Granicus only. |
| `sf_housing_pipeline.py` | AB 2011 / SB 35 / density bonus housing pipeline | Socrata REST | Cross-refs parcels dataset for district. Watchlist tracks 400 Divisadero. State-tracked. |
| `sf_journalism.py` | News aggregator — 5 SF outlets | RSS | Mission Local, Streetsblog SF, SF YIMBY, 48 Hills, SF Standard. SF Examiner 404s. Archive: `sf_journalism_archive.json`. |
| `sf_volunteer_cleanups.py` | Community cleanup events | curl/JS bundle | Refuse Refuse SF + DPW Love Our City NBDs. GoDaddy SPA — data is in JS bundle, not HTML. |
| `sf_bart_board.py` | BART Board of Directors + advisory bodies | Legistar API | `webapi.legistar.com/v1/bart/Events`. Board = HIGH, advisory = MEDIUM. `--next` for next meeting. |
| `sf_sfusd_board.py` | SFUSD Board of Education meetings | Google Sheets + BoardDocs | 2nd & 4th Tuesdays 5pm. Falls back to projected schedule if sources unavailable. |
| `sf_evictions.py` | Eviction notices by district | Socrata REST | Dataset `5cei-gny5`. Ellis Act + OMI = displacement signals. Trend comparison (two windows). `--district N`. |
| `sf_sfcta.py` | SF County Transportation Authority | curl (sfcta.org) | Congestion pricing, Muni Forward, bike/ped funding, Prop L sales tax. Board + CAC + committees. |
| `sf_civic_actions.py` | Protests, rallies, civic actions | Mobilize.us API + Indybay | No auth. Rallies/marches = HIGH, canvass/phone bank = MEDIUM. `--type rally` to filter. |
| `sf_mission_local.py` | Mission Local news only | RSS | Subset of sf_journalism.py. Use sf_journalism.py for multi-outlet. |
| `config_loader.py` | District config loader | — | All 11 districts pre-populated. Pass `--district N` to any script. |

### No browser required
All scripts now work without a browser. `sf_board_of_appeals.py` and `sf_rent_board.py` use the sf.gov Wagtail CMS API (`api.sf.gov/api/v2/pages/`). `sf_planning_commission.py` uses curl against server-rendered Drupal HTML (the sfplanning.org calendar pages are not JS-dependent despite initial assumptions).

---

## District Configuration

All scripts accept `--district N` (1–11). That's all you need for a basic run.

User preferences and neighborhood context (streets, addresses, people to flag) live in **`USER.md`** — a human-readable profile file. There is no `civic_config.json`; that file has been removed. Read `USER.md` to understand what the user cares about before writing any report.

District→neighborhood mappings: `references/sf-districts.md`
Build timeline benchmarks: `references/sf-build-timelines.md` — **Read before writing about any housing project.** Includes typical phase durations, financing tracking methods, and context for whether a project is on track or stalled.

---

## Report Generation

### Agent-driven (recommended for one-off reports)

```bash
# 1. Collect data
python3 scripts/sf_weekly_digest.py --json > /tmp/civic_data.json
python3 scripts/sf_housing_pipeline.py --district 5 --json >> /tmp/civic_data.json
python3 scripts/sf_volunteer_cleanups.py --district 5 --days 14 --json >> /tmp/civic_data.json

# 2. Agent synthesizes following STYLE.md
# Read STYLE.md, read the JSON, write narrative to reports/
```

### Agent-driven reports (quarterly, annual, deep dives)

Give your agent the workspace path and let it orchestrate:

> Read STYLE.md and USER.md. Run the scripts in scripts/ to collect data for District 5, then write a Q1 2026 civic digest. Save to reports/.

**⚠️ Path gotcha:** Agents default to wrong relative paths. Always resolve the absolute path to `scripts/` from the location of this SKILL.md before running anything. Use full absolute paths — e.g. `python3 /full/path/to/scripts/sf_housing_pipeline.py`. Never just `scripts/SCRIPTNAME.py`.

### Report style
Read `STYLE.md` before writing any report. Key rules:
- TLDR (5-7 bullets) first — last bullet is always one action
- Officials box: YOUR elected officials only (supervisor, mayor, state reps)
- Short sections (3-5 sentences), scan-and-skip
- Connect threads across sources (lobbying + permits + 311 + journalism = one story)
- End each thread with a specific action (date, email address, what to say)
- Rates over counts. "Up 15%" beats "95 reports."
- Tone: pro-building, pro-tech, car-free lens. See STYLE.md for full framing guidance.

Reference: see `STYLE.md` for rules and examples.

---

## Key Data Gotchas

These will burn you if you don't know them:

**Socrata column names:**
- 311 (`vw6y-z8j6`): `supervisor_district` = `'5.0'` (float string), date field = `requested_datetime`
- Building Permits (`i98e-djp9`): `supervisor_district` = `'5'`. Address split across `street_number`/`street_name`/`street_suffix` — no combined field. `estimated_cost` is a string, cast in Python.
- Lobbyist Contacts (`5f5n-tdbf`): NO underscores — `officialname`, `clientname`, `lobbyistname`
- Lobbyist Contributions (`e6py-fg8b`): same — `lobbyistname`, `candidatename`, `sourceoffunds`
- Planning Records (`qvu5-m3a2`): `pim_link` returns `{"url": "..."}` dict, not string. No `supervisor_district` — cross-reference parcels (`acdm-wktn`) by `block`/`lot`.

**Legistar:** The API is broken for SF (returns "Agenda Draft Status" error). Use the RSS feed instead:
`https://sfgov.legistar.com/Feed.ashx?M=C&ID=17442&GUID=EEE85B7C-1A1C-4A56-873E-355A0A0DE5C3&Mode=All&Format=rss`
Returns ~100 meetings (~6 months). Parse meeting IDs from RSS, then `fetch_meeting_detail()` for agenda items.

**Reddit:** Direct curl blocked (403, datacenter IP). Use `web_fetch` tool — routes differently, works.

**sfrecpark.org:** Completely unreachable from this server. Granicus (`view_id=91`) is the only path.

**311 trend pattern:** Query two 7-day windows (current + prior week) and show week-over-week delta. "Encampments up 15%" is useful. "95 encampments" is not.

**Journalism archive:** `sf_journalism.py` persists articles to `sf_journalism_archive.json` (cap 2000). For quarterly history, use `--archive --days 90`.

---

## Socrata Dataset Reference

| Dataset | ID | Key fields |
|---------|-----|-----------|
| Building Permits | `i98e-djp9` | permit_number, permit_type, address (split), supervisor_district ('5'), estimated_cost (string) |
| 311 Cases | `vw6y-z8j6` | service_name, requested_datetime, supervisor_district ('5.0') |
| Planning Dept Records | `qvu5-m3a2` | record_id, description, record_status, project_address, block, lot, number_of_units_net, pim_link (dict) |
| SF Parcels | `acdm-wktn` | block_num, lot_num, supervisor_district, zoning_code |
| Lobbyist Contacts | `5f5n-tdbf` | officialname, lobbyistname, clientname, outcomesought, subjectarea, filenumber |
| Lobbyist Contributions | `e6py-fg8b` | lobbyistname, candidatename, sourceoffunds, committeename |
| Assessor Parcels | `fk72-cxc3` | property_location, parcel_number, assessed_land_value, zoning_code |
| Eviction Notices | `5cei-gny5` | eviction_id, address, neighborhood, file_date, supervisor_district, ellis_act_withdrawal, owner_move_in |

Query pattern: `https://data.sfgov.org/resource/{id}.json?$where=supervisor_district='5'&$order=date+DESC&$limit=50`

---

## Scraping Method Reference

| Site | curl works? | Browser needed? |
|------|-------------|-----------------|
| Legistar (sfgov.legistar.com) | ✅ | No |
| SFMTA (sfmta.com) | ✅ | No |
| SF Planning notices (sfplanning.org/notices) | ✅ (needs Accept header) | No |
| SF Planning calendars (sfplanning.org/hearings-*) | ✅ (use `-list` views) | No |
| SF.gov meetings (api.sf.gov) | ✅ (Wagtail API) | No |
| Socrata (data.sfgov.org) | ✅ | No |
| SF Ethics (sfethics.org) | ✅ | No |
| Mission Local / journalism RSS | ✅ | No |
| Granicus (Rec & Park) | ✅ | No |
| Refuse Refuse SF (refuserefusesf.org) | ✅ (JS bundle) | No |
| sfrb.org | ✅ (Wagtail API via api.sf.gov) | No |

---

## Scheduling

### Weekly digest (Monday morning)
```json
{
  "schedule": { "kind": "cron", "expr": "0 14 * * 1", "tz": "America/Los_Angeles" },
  "payload": { "kind": "systemEvent", "text": "Run SF civic weekly digest and send summary" }
}
```

### Daily delta (weekday noon check-in)
```json
{
  "schedule": { "kind": "cron", "expr": "0 12 * * 1-5", "tz": "America/Los_Angeles" },
  "payload": { "kind": "systemEvent", "text": "Run SF civic daily delta and report any new agenda items" }
}
```

---

## File Layout

```
skills/sf-civic-digest/
├── SKILL.md              ← you are here
├── STYLE.md              ← report style guide (read before writing any report)
├── USER.md               ← user profile: district, neighborhoods, streets, interests (fill this in)
├── .gitignore            ← excludes runtime state from version control
├── scripts/              ← all data pipeline scripts
│   ├── sf_weekly_digest.py      ← master aggregator
│   ├── sf_civic_digest.py       ← Board of Supervisors (Legistar)
│   ├── sfmta_hearings.py        ← SFMTA Engineering Hearings
│   ├── sf_planning_notices.py   ← Planning project notices
│   ├── sf_planning_commission.py← Planning Commission / HPC / ZA
│   ├── sf_board_of_appeals.py   ← Board of Appeals
│   ├── sf_building_permits.py   ← Building permits (Socrata)
│   ├── sfgov_events.py          ← Supervisor events
│   ├── sf_311.py                ← 311 service requests (Socrata)
│   ├── sf_ethics.py             ← Lobbyist contacts + contributions
│   ├── sf_rent_board.py         ← Rent Board
│   ├── sf_sfmta_board.py        ← SFMTA Board of Directors
│   ├── sf_rec_park.py           ← Rec & Park Commission
│   ├── sf_housing_pipeline.py   ← Housing pipeline (AB2011/SB35/density bonus)
│   ├── sf_journalism.py         ← News aggregator (5 outlets)
│   ├── sf_volunteer_cleanups.py ← Community cleanups (Refuse Refuse + DPW)
│   ├── sf_evictions.py          ← Eviction notices (Socrata)
│   ├── sf_bart_board.py         ← BART Board of Directors (Legistar)
│   ├── sf_sfusd_board.py        ← SFUSD Board of Education
│   ├── sf_sfcta.py              ← SF County Transportation Authority
│   └── config_loader.py         ← District config loader
├── reports/              ← generated reports
│   └── archive/          ← older versions
└── references/
    ├── sf-districts.md       ← district map, supervisor names, neighborhood lists
    └── sf-build-timelines.md ← build phase benchmarks, financing signals, project context
```

---

## Participation Guide

### How to comment at SFMTA Engineering Hearings
- Online: join at `sfmta.com/EngHearing` when your item is called, raise hand
- Phone: number + conference ID in hearing notice
- Written: email the contact listed per item with "Public Hearing" in subject
- Decisions posted the following Friday at `sfmta.com/EngineeringResults`

### How to comment at Planning Commission / HPC
- Written comments accepted before each hearing via sfplanning.org
- HPC meets 1st and 3rd Wednesdays (low attendance, high impact)
- In-person: City Hall Room 400

### How to reach your supervisor
- Mahmood (D5): bilal.mahmood@sfgov.org
- Constituent meetings: request via supervisor's website
- Board public comment: first Tuesday of each month, City Hall Room 250
