# Monday drafts — scheduled task

- **Name:** CSM feedback drafts
- **Schedule:** `CRON_TZ=Europe/Madrid 42 7 * * 1` (every Monday 7:42, drafts ready before 8:00)
- **Status:** disabled for now (re-enable from the scheduled task settings in Claude)
- **Runs:** in the cloud, automatic approval, push notification at the end
- **Writes to:** the artifact database, collection `drafts` — never posts to Jira/Slack/email

## Prompt

```
Prepare this week's mid-flight feedback DRAFTS for Enzo (Operations Market Lead BNL at Seedtag) and store them in his "CSM Feedback Generator" artifact (slug <ARTIFACT_SLUG>). Never post anything to Jira, Slack or email — drafts only. Work unattended; don't ask questions.

0. SELECTION. With the artifacts connector, db_get path "config/watchlist" → field "crms" (list of CRM IDs selected). Draft ONLY these campaigns, and only those still live (end date >= today). If the list is missing or empty, write nothing and finish with "No campaigns selected for Monday drafts."
1. DATA per selected CRM ID. With business-data-cube-mcp (run_query), last 14 days up to yesterday (Europe/Madrid), day granularity, filter salesforce_crm_id:
   - cube direct_product_line_delivery_tz (Direct): product_line_name, am (= CSM), adops (= ops), advertiser, payment_method, booked_revenue, first_kpi, first_kpi_value (fraction: 0.006 = 0.6%), start_dt, end_dt, days_left, salesforce_configured_units, units_accumulated, units_goal (daily goal), units_pacing_ratio (pacing % to date), ctr_accumulated / vw_accumulated / vtr_accumulated (already in %), daily impressions/units_delivered/clicks/ctr/vw/vtr.
   - if not found there: cube pmp_product_line_delivery_tz (PMP): salesforce_configured_impressions, impressions_accumulated, impressions_goal, impressions_pacing_ratio, adserver_*_accumulated, gross_revenue_spent_ratio.
   - if in neither (performance on DV360): Jira (Atlassian connector, cloudId f27c696c-ab8c-4c73-896e-079ad4bb1763) `project = PT AND cf[11704] ~ "<id>"` → PT key; Google Drive spreadsheet whose title starts with "<PT key>__" (ignore _REPORTING/_OPTIMIZATION); Google Sheets: input_jira!A1:B10 and dCPX!A1:S502 (Budget, Daily advertiser target, "Total pacing index"/"Yesterday pacing" and label, Total row and daily rows: Impressions, Clicks, Client's spent, Client's CPC/CPM, KPI = visits, CPX = cost per visit). NEVER use or mention media cost or margin.
2. JIRA BNL: `project = BNL AND (summary ~ "\"<id>\"" OR cf[11704] ~ "<id>")` → ticket key, Second KPI (customfield_11783 / customfield_11784), CSM (customfield_12217) if missing.
3. STATUS: UNDER PACING if below the daily goal 3+ consecutive days up to yesterday (performance: client spend below the daily target); TO WATCH if pacing < 95% or a KPI below target; else ON TRACK. Projections use the last-3-days average. Direct never delivers above 100% of booking; PMP often runs ahead early and ends ~90–95% of budget because of fees (normal).
4. PREFERENCES: db_get "style/rules" (apply every rule whose "on" is not false; they override the defaults below) and db_list collection "examples" (3 most recent texts: match their voice, not their numbers).
5. WRITE one consolidated draft per CSM + advertiser (one section per CRM ID), as a MID-FLIGHT UPDATE, "Factual" style, English, Jira markdown (**bold** only for key figures):
   - A REAL FEEDBACK, not a data dump: for each campaign say whether it's going well, why, and what it means. Every figure quoted must be interpreted. Max 5 figures per campaign.
   - Human and natural; no "Overall", "It's worth noting", "Additionally", "Furthermore", "ensure", "leverage"; no emojis.
   - Greet the CSM by first name, one-sentence summary, then per campaign a header **CRM_ID · short campaign name — status**.
   - NO recommendations, never suggest changing creatives. Only real figures. CTR 2 decimals, other % 1 decimal, thousands separated by spaces. Use "VTR" for video completion. NO signature.
6. STORE: batch = today's date (YYYY-MM-DD). One db_write per draft (wait 2 s between calls), op "set", path "drafts/<batch>_<first CRM ID>", data {"ts": <epoch ms>, "batch": "<batch>", "crms": ["<id>", ...] (strings), "csm": "<CSM full name>", "ops": "<adops full name>", "label": "<advertiser> (<n> campaign(s))", "flag": true if any campaign is UNDER PACING or TO WATCH, "status": "new", "mode": "update", "text": "<draft>"}. Skip a path that already exists.
7. Finish with a short summary: number of drafts and flagged campaigns (CRM · advertiser · reason).
```
