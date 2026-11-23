# Monday drafts — scheduled task

- **Name:** CSM feedback drafts
- **Schedule:** `CRON_TZ=Europe/Madrid 42 7 * * 1` (every Monday 7:42, drafts ready before 8:00)
- **Status:** enabled (routine "CSM feedback drafts" on claude.ai, Cowork scheduled task)
- **Runs:** in the cloud, automatic approval, push notification at the end
- **Writes to:** the artifact database, collection `drafts` — never posts to Jira/Slack/email

## Prompt

```
Prepare this week's mid-flight feedback DRAFTS for Enzo (Operations Market Lead BNL at Seedtag) and store them in his "CSM Feedback Generator" artifact (slug <ARTIFACT_SLUG>). Never post anything to Jira, Slack or email — drafts only. Work unattended; don't ask questions.

0. SELECTION. With the artifacts connector, db_get path "config/watchlist" → field "crms" (list of CRM IDs selected). Draft ONLY these campaigns, and only those still live (end date >= today). If the list is missing or empty, write nothing and finish with "No campaigns selected for Monday drafts."
1. DATA per selected CRM ID. With business-data-cube-mcp (run_query), last 14 days up to yesterday (Europe/Madrid), day granularity, filter salesforce_crm_id:
   - cube direct_product_line_delivery_tz (Direct): product_line_name, am (= CSM), adops (= ops), advertiser, payment_method, booked_revenue, first_kpi, first_kpi_value (fraction: 0.006 = 0.6%), start_dt, end_dt, days_left, salesforce_configured_units, units_accumulated, units_goal (daily goal), units_pacing_ratio (pacing % to date), ctr_accumulated / vw_accumulated / vtr_accumulated (already in %), daily impressions/units_delivered/clicks/ctr/vw/vtr.
   - if not found there: cube pmp_product_line_delivery_tz (PMP): salesforce_configured_impressions, impressions_accumulated, impressions_goal, impressions_pacing_ratio, adserver_*_accumulated, gross_revenue_spent_ratio.
   - if in neither (performance on DV360): Jira (Atlassian connector, cloudId f27c696c-ab8c-4c73-896e-079ad4bb1763) `project = PT AND cf[11704] ~ "<id>"` → PT key; Google Drive spreadsheet whose title starts with "<PT key>__" (ignore _REPORTING/_OPTIMIZATION); Google Sheets: input_jira!A1:B10 (price type, price, dates, budget, KPI, KPI value) and dCPX!A1:S502 (Budget, Daily advertiser target, "Total pacing index"/"Yesterday pacing" and label, Total row and daily rows: Impressions, Clicks, Client's spent, Client's CPC/CPM, KPI = visits, CPX = cost per visit). NEVER use or mention media cost or margin.
   - Buying Engine strategies: cube buying_engine_deliveries_utc, dimension strategy_name, measures impressions, clicks, viewable_impressions, view_measured_impressions, video_starts, video_completions, filter salesforce_crm_id, dt_utc from start date to yesterday (max 14 days per query: chunk the range).
2. JIRA BNL: `project = BNL AND (summary ~ "\"<id>\"" OR cf[11704] ~ "<id>")` → ticket key, Second KPI (customfield_11783 / customfield_11784), CSM (customfield_12217) if missing.
3. STATUS: UNDER PACING if below the daily goal 3+ consecutive days up to yesterday (performance: client spend below the daily target); TO WATCH if pacing < 95% or a KPI below target (performance: report says under/over pacing, or KPI off target); else ON TRACK. Projections use the last-3-days average. Direct never delivers above 100% of booking; PMP often runs ahead early and ends ~90–95% of budget because of fees (normal).
4. PREFERENCES: db_get "style/rules" (apply every rule whose "on" is not false; they override the defaults below) and db_list collection "examples" (up to 5: those with "manual": true first, then the most recent "update" ones). They are the VOICE REFERENCE: copy their sentence rhythm, openings, connectors, vocabulary and informality; follow them over the style defaults below; never reuse their numbers.
5. WRITE one consolidated draft per CSM + advertiser (one section per CRM ID), as a MID-FLIGHT UPDATE, "Factual" style, English, Jira markdown (**bold** only for key figures):
   - A REAL FEEDBACK, not a data dump: for each campaign say whether it's going well, why, and what it means. Every figure quoted must be interpreted. Max 5 figures per campaign.
   - Human and natural, like a quick comment to a colleague; contractions. Vary the shape: don't open two sections or two sentences in a row the same way; link ideas with natural connectors ("so", "that said", "which means", "for now"), one or two per section. Never open a sentence with "One campaign", "The other campaign", "This campaign" or "The campaign": use the advertiser or product name, or start with the fact.
   - Never: "Overall", "It's worth noting", "Notably", "In summary", "Additionally", "Furthermore", "Moreover", "ensure", "leverage", "showcase", "highlight", "robust", "remarkable", "impressive", "solid/strong performance", "healthy", "moving forward", "going forward", "in terms of", "continues to perform well", "Let me know if you have any questions"; no emojis, no em-dash chains.
   - End with ONE short forward-looking line (what we're watching this week or when the next update comes), never a recap or a formula.
   - Greet the CSM by first name, one-sentence summary, then per campaign a header **CRM_ID · short campaign name — status**.
   - If the campaign has Buying Engine strategies, ALWAYS add "**Strategies:**" with one bullet per strategy, biggest first: "- **<name>**: <share>% of impressions · <first KPI metric: VTR, CTR or viewability> <value>", then one sentence on which carries the volume and which performs best. Mandatory, whatever the preferences say about leaving out metrics.
   - NO recommendations, never suggest changing creatives. Only real figures. CTR 2 decimals, other % 1 decimal, thousands separated by spaces. Use "VTR" for video completion. NO signature.
6. STORE: batch = today's date (YYYY-MM-DD). One db_write per draft (wait 2 s between calls), op "set", path "drafts/<batch>_<first CRM ID>", data {"ts": <epoch ms>, "batch": "<batch>", "crms": ["<id>", ...] (strings), "csm": "<CSM full name>", "ops": "<adops full name>", "label": "<advertiser> (<n> campaign(s))", "flag": true if any campaign is UNDER PACING or TO WATCH, "status": "new", "mode": "update", "text": "<draft>"}. Skip a path that already exists.
7. Finish with a short summary: number of drafts and flagged campaigns (CRM · advertiser · reason).
```
