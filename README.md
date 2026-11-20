# CSM Feedback Generator — BNL Ops · Seedtag

A single-page tool (Claude artifact) that turns one or more **CRM IDs** into a ready-to-post **campaign feedback for the CSM**: it reads delivery and KPIs from BigQuery, the booking from Jira, the performance internal report from Google Sheets and targeting insights from the data lake, then asks Claude to write a human, "real" feedback (not a data dump) that is posted as a Jira comment.

Owner: Enzo (Operations Market Lead BNL). Live artifact: https://claude.ai/artifact/5Fe2486gYwq4stpRzuacs8 (shared with the Seedtag org).

---

## 1. Features

| Tab | What it does |
|---|---|
| **Feedback** | CRM IDs (chips) · *Browse live campaigns* (all live BE+NL campaigns, filter by ops / CSM / "to watch") · feedback type (Auto / Mid-flight update / End-of-campaign report) · focus period (default whole flight) · extra KPI files (PNG, JPG, Excel, CSV) · free context · per-campaign cards (status + reason, KPI vs target, daily chart vs daily goal, projection & budget at risk, since-last-update deltas, flight insights) · generated feedback with styles **Factual** (default) / **Short** / **Detailed**, Edit/Preview, select-a-passage comment bubble (Fix + "Remember"), Learn from my edits, Regenerate, Copy, **Export PDF**, **Post as Jira comment** (with @mention of the CSM) · History (5 latest, "show all"; a saved feedback reopens editable, copyable, exportable and postable) |
| **Monday drafts** | Selection of CRM IDs to draft every Monday (add / remove) · list of drafts (flagged first, ops filter, delete one / posted / all) · click a draft → opens it in Feedback with refreshed figures |
| **My preferences** | Style rules applied to every feedback (toggle, edit by double-click, delete, add) · approved examples (feedbacks posted to Jira; the 3 latest are used as style models; deletable) |

### Feedback types
- **Mid-flight update** — concise, factual, **no recommendations**.
- **End-of-campaign report** — whole flight: final delivery vs booking, KPIs vs targets, evolution, **Where it performed** (bullets: Contexts / Publishers / Devices / Strategies), **Recommendations for next time**.
- **Auto** — end report if every selected campaign has ended, update otherwise.

### Styles
- **Factual** (default) — one bullet per metric "Metric: value vs target → verdict", then one conclusion sentence.
- **Short** — 1–2 sentences per campaign, max 2 figures.
- **Detailed** — a short narrative paragraph per campaign, max 5 figures, each interpreted.

---

## 2. Data sources

| Source | Connector (display name) | Used for |
|---|---|---|
| Semantic layer on BigQuery | `business-data-cube-mcp` → `run_query` | Direct & PMP delivery, pacing, KPIs, live campaigns, Buying Engine strategies |
| Jira | `Atlassian Rovo` → `searchJiraIssuesUsingJql`, `addCommentToJiraIssue` | Booking ticket (BNL), performance ticket (PT), KPIs, CSM, posting |
| Google Drive | `Google Drive` → `search_files` | Finding the performance internal report |
| Google Sheets | `Google Sheets` → `get_values` | Reading the internal report |
| Data lake (Trino) | `adx-data-toolbox` → `trino_query` | Contexts / publishers / devices (post-campaign analysis) |

Jira cloudId: `f27c696c-ab8c-4c73-896e-079ad4bb1763` (seedtag.atlassian.net).

### 2.1 Cubes (business-data-cube-mcp)
- `direct_product_line_delivery_tz` — **Direct** product line (CRM ID) per day: `salesforce_crm_id`, `product_line_name`, `am` (= CSM), `adops` (= ops), `advertiser`, `agency`, `product`, `payment_method`, `payment_value`, `booked_revenue`, `currency`, `first_kpi`, `first_kpi_value` (**fraction**, 0.006 = 0.6%), `start_dt`, `end_dt`, `days_left`, `salesforce_configured_units`, `units_accumulated`, `units_goal` (daily goal), `units_pacing_ratio` (pacing % to date), `ctr_accumulated` / `vw_accumulated` / `vtr_accumulated` (**already %**), `budget_at_risk`, measures `impressions`, `clicks`, `units_delivered`, `ctr`, `vtr`, `vw`.
  - ⚠️ `salesforce_units_delivered_ratio_projected` is a **month-end** projection — not used.
- `pmp_product_line_delivery_tz` — **PMP**: same idea with `salesforce_configured_impressions`, `impressions_accumulated`, `impressions_goal`, `impressions_pacing_ratio`, `adserver_*_accumulated`, `gross_revenue_spent_ratio`.
- `direct_flight_delivery_tz` — `external_execution_id` per CRM ID (to query the data lake).
- `buying_engine_deliveries_utc` — **strategies** (`strategy_name`) with impressions, clicks, viewable/measured, video starts/completions. Max **14 days per query** → the tool queries the flight in 14-day chunks (max 8).
- Delivery history window: **365 days** (`HISTORY_DAYS`), CRM IDs queried in batches of 10 (`CRM_BATCH`) to stay under the 5 000-row limit. For a flight that started earlier, cumulative totals and pacing still cover the whole flight; the daily chart and weekly insights start 365 days ago (flagged on the campaign card and in the prompt).
- Dates ("today", "yesterday") are computed in **Europe/Madrid**.

### 2.2 Jira
- **BNL AdOps - Bookings** (`project = BNL`): summary starts with the CRM ID (`293425 | …`).
  - `customfield_11704` CRM ID · `customfield_11781`/`11782` First KPI name/value · `customfield_11783`/`11784` Second KPI · `customfield_12217` CSM (user, `accountId` used for @mention) · `customfield_11759` booked impressions · `customfield_10901` start · `duedate` end · `customfield_12051` Ad Manager link.
- **Performance Traders** (`project = PT`): summary does **not** contain the CRM ID → search `cf[11704] ~ "<id>"`.
- Search used: `project in (BNL, PT) AND (summary ~ "\"<id>\"" OR cf[11704] ~ "<id>")`.

### 2.3 Performance internal report (Google Sheets)
- Performance campaigns on **DV360** are **not** in the cubes.
- File = spreadsheet whose title starts with `<PT key>__` (e.g. `PT-7965__Emirates__…`); ignore `PT-XXXX_REPORTING` / `_OPTIMIZATION`.
- Tabs read: `input_jira!A1:B10` (price type, price, dates, budget, KPI, KPI value), `dCPX!A1:S502` (Budget, Daily advertiser target, "Total pacing index", "Yesterday pacing" + label, Total row, daily rows: Impressions, Clicks, **Client's spent**, Client's CPC/CPM, KPI = visits, CPX = cost per visit), `metrics!A1:AB80` (tactics breakdown).
- **Never** use or show media cost / margin.

### 2.4 Targeting insights (data lake)
Table `st_datalakehouse.lab.post_campaign_analysis_view_v2`, filtered by `execution_id` (from `direct_flight_delivery_tz.external_execution_id`) and date. Aggregated by `result_context_categories_tier1` (context), domain of `canonical_url` (publisher), `device`.
- CTR = `ad_click / ad_impression` · VTR = `video_complete / video_play` · Viewability = `ad_view_1s50 / ad_view` (display) or `ad_view_2s50 / ad_view` (video).
- Only Seedtag-served campaigns (direct / Buying Engine). Not available for DV360 performance.

---

## 3. Business rules

- **Under pacing** = delivered below the daily goal for **N consecutive days** (default 3, setting).
- **To watch** = pacing < 95% or a KPI below target (performance: internal report says under/over pacing, or KPI off target).
- **Projection / budget at risk** = (delivered + days left × **average of the last 3 days**) / booked; budget at risk = booked revenue × (1 − projection).
- **Direct** never delivers above 100% of booking (projection capped at 100%). **PMP** can run ahead early and usually ends around **90–95% of the budget** (fees) — normal.
- Mid-flight updates: **no recommendations**. End reports: recos from 3 levers only — (1) capitalise on what worked (strategies, contexts, publishers, devices), (2) add formats (video/CTV if display, display/CTV if video), (3) more budget next time **only** if delivery went fine. **Never recommend changing or adding creatives.**
- No signature (Jira shows the author). The feedback is written as `AUTHOR` (constant at the top of the script, "Enzo"). English. Jira markdown.
- Each data source can fail on its own: if BigQuery, Jira, a Gsheet or the data lake doesn't answer, the tool keeps going with the others and shows a warning (a CRM ID with no data because BigQuery was down is shown as **BIGQUERY DOWN**, not NO DATA).
- UI colours: Seedtag palette only (white, cream, coral, black, 5 greys). Statuses: on track = black, to watch = coral outline, under pacing = coral, not started = light grey, ended = dark grey.
- Default preferences: see `config/default-preferences.json`.

---

## 4. Artifact database (owner-only)

| Path | Content |
|---|---|
| `feedbacks/<id>` | History: `ts`, `crms`, `label`, `feedback`, `mode`, `jira` (per-CRM `key`, `url`, `csm`, `csmId`, so a reopened feedback can be posted with the @mention), `statuses`, `snap` (per-CRM snapshot used for "since last update"), `posted` |
| `examples/<id>` | Approved feedbacks (saved when posted): `ts`, `mode`, `crms`, `text` |
| `style/rules` | `{ rules: [{ text, on, ts }] }` — My preferences |
| `drafts/<batch>_<crm>` | Monday drafts: `ts`, `batch`, `crms`, `csm`, `ops`, `label`, `flag`, `status` (new/posted), `mode`, `text` |
| `config/watchlist` | `{ crms: [...] }` — campaigns to draft on Mondays |

---

## 5. Publishing / editing the artifact

The page is one self-contained `index.html` (CSS + JS inline; libraries from cdnjs: SheetJS, html2canvas, jsPDF).

1. Edit `index.html`, commit and push to this repo.
2. Republish the file to the live artifact (https://claude.ai/artifact/5Fe2486gYwq4stpRzuacs8) from Claude with the Artifact tool, passing that URL so the link stays the same. The artifact is shared with the Seedtag org: viewers get the update immediately.
3. Capabilities are stored with the artifact and kept on republish. `config/capabilities.json` mirrors them; when adding a capability, publish with the new full declaration.
4. Connector names at the top of the script (`CUBE_SRV`, `JIRA_SRV`, `DRIVE_SRV`, `SHEETS_SRV`, `ADX_SRV`) must match the **display names** of the connectors in the viewer's Claude.

### Working with someone outside the Claude team
- An artifact can only be edited by people of the owner's Claude organization. A collaborator from another Claude team works through this repo: open a PR on `index.html`, then the owner republishes; or the collaborator publishes their **own copy** in their Claude (their own artifact, own database, own connectors).
- The connectors use the **viewer's** credentials: a collaborator needs access to the same data (Seedtag cube, Jira, Drive, data lake) and the same connector display names (or adapt the constants).

### Collaboration workflow
This repo is the single source of truth for the code. Each published artifact (the owner's, and any collaborator's copy) is a separate deployment with its **own database** (history, preferences, drafts): changes in one artifact never reach the repo or another artifact on their own.

1. **Never commit to `main` directly.** Create a branch, commit, open a pull request. Recommended: protect `main` (*Settings → Branches → Add rule → Require a pull request before merging*).
2. **Keep PRs small and merge them quickly** — almost everything lives in one `index.html`, so long-lived branches conflict. Say who is working on which part (prompt, data fetch, UI…).
3. **Pull/download the latest `main` before starting** a change.
4. **After a merge, each person republishes their own artifact** from `main` (same URL, same capabilities from `config/capabilities.json`). Nothing republishes automatically.
5. **Shared artifact (one URL, shared data):** only possible when both people are in the same Claude organization. The owner then shares the artifact with edit access, and the `db` rules in `config/capabilities.json` must be relaxed from `owner` to `admin` for the paths that should be shared.

---

## 6. Scheduled task (Monday drafts)
See `scheduled-task/monday-drafts-prompt.md`. Currently **disabled**.

## 7. Changelog

**2026-11-20**
- Prompt: removed contradictions (bullets vs paragraphs now follow the chosen style; metrics with no value or no target are left out).
- Data loading keeps going when a source fails (BigQuery, Jira, Gsheet, data lake) and shows the warnings after the results.
- Dates in Europe/Madrid instead of UTC; no more `NaN` projection on performance campaigns without spend; history window 180 → 365 days, with the limit shown in Settings and on the card.
- Targeting insights and Buying Engine strategies fetched in parallel.
- History: a saved feedback reopens editable, copyable, exportable and postable on its Jira tickets.
- Removed the Signature setting (author name is the `AUTHOR` constant).
- Seedtag palette only (no more green / amber / blue).
- PDF export: pages break between paragraphs, table rows and campaign cards (a card that fits on a page is never split); wide tables wrap; very long reports keep their end (resolution lowered instead of truncated).

**v1** — first version.

## 8. Repo layout
```
index.html                         the artifact page
config/capabilities.json           capability declaration (connectors, Claude, db rules, downloads)
config/default-preferences.json    style rules currently in My preferences
scheduled-task/monday-drafts-prompt.md
```
