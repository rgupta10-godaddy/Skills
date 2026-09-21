## Shared Constants and Field Mappings

### Scope

This skill contains shared constants, field IDs, aliases, URLs, defaults, Helix sizing table references, timing rules, ratio metric definitions, channel abbreviations, scorecard fallback references, and workflow-specific defaults.

### Dependency-only execution contract

This skill is dependency-only.

It must not:
- Invoke `view_skill`.
- Open another skill.
- Route to another workflow gate.
- Retrieve Jira, Confluence, Hivemind, or Helix data.
- Ask the user questions.
- End the selected workflow.

It must:
- Return constants and mappings only.
- Return control to the caller immediately after constants are available.
- Preserve workflow_intent.
- Preserve current_gate.

When consulted during opportunity sizing:
- Set sizing_shared_constants_loaded = true.
- Do not reload this skill again in the same sizing workflow unless the user switches Jira tickets or explicitly restarts sizing.
- Loading constants or mappings is never a terminal workflow action.

### Jira fields

**Source:** Confluence - Commonly Used Jira Custom Fields
- **Page:** https://godaddy-corp.atlassian.net/wiki/spaces/REN/pages/4559275097
- **Page ID:** 4559275097
- **Cloud ID:** godaddy-corp.atlassian.net

**Usage:**
On first invocation of this skill in a workflow session:
1. Fetch the Jira field mapping table from Confluence using mcp__atlassian__getConfluencePage with pageId="4559275097" and contentFormat="markdown"
2. Parse the markdown table to extract field mappings
3. Cache the field IDs in workflow state for use by all skills in this session
4. If Confluence fetch fails, fall back to critical field IDs below

**Critical Jira field IDs (fallback when Confluence unavailable):**
- Hypothesis: customfield_24203
- Hivemind link: customfield_34621
- Channels: customfield_24501
- Variants: customfield_17203
- Financial impact: customfield_24800
- End date: customfield_14200
- Outcome: customfield_17001

### Test Duration field (customfield_16515) — cached allowed values

**Source:** live Jira field metadata query, project CM, issue type Story, verified 2026-09-03. Option IDs confirmed identical on project REN, issue type Story, verified 2026-09-18 — the table below is valid for both projects.

Field schema: single-select option field (`com.atlassian.jira.plugin.system.customfieldtypes:select`).

| Value | Option ID |
|---|---|
| 3 days | 16740 |
| 1 week | 16741 |
| 10 days | 33916 |
| 2 weeks | 16742 |
| 3 weeks | 16743 |
| 4 weeks | 16744 |
| 5 weeks | 16745 |
| 6 weeks | 16746 |
| 7 weeks | 33917 |
| 8 weeks | 33918 |
| 9 weeks | 33919 |
| 10 weeks | 33920 |
| 11 weeks | 33921 |
| 12 weeks | 33922 |
| 13 weeks | 33923 |
| 14 weeks | 33924 |
| 15 weeks | 33925 |
| 16 weeks | 33926 |
| 17 weeks | 33927 |
| 18 weeks | 33928 |
| 19 weeks | 33929 |
| 20 weeks | 33930 |
| 21 weeks | 33931 |
| 22 weeks | 33932 |
| 23 weeks | 33933 |
| 24 weeks | 33934 |
| 25+ weeks | 33935 |

`eor-renewal-experiment-worfklow-confluence-publish-and-jira-link.md` uses this table as the default source for the closest-option mapping instead of a live Jira metadata query on every run. See that skill's "Jira field value discovery for customfield_16515" section for exactly when a live query is still required (cache miss, write failure, or a project other than CM or REN).

### Financial impact field (customfield_24800) — type note

**Source:** live Jira field metadata query, project CM, issue type Story, verified 2026-09-03.

Field schema: `number` (`com.atlassian.jira.plugin.system.customfieldtypes:float`).

This field only accepts a native JSON number. Do not send a string, and do not include a currency symbol, commas, or other formatting characters in the value sent to Jira. See `eor-renewal-experiment-worfklow-confluence-publish-and-jira-link.md` for the writeback rule.

### Helix fields for sizing only

These fields apply only to the opportunity sizing workflow.

#### Renewal sizing fields

| Concept | Table / field |
|---|---|
| Sizing table | ba_commercial_success.eor_sizing_renewals |
| Product dimension lookup table | bigreporting.dim_product_snap |
| Shopper ID | prior_bill_shopper_id |
| Prior-bill product PNL line | prior_bill_product_pnl_line_name |
| Auto-renew status | historical_auto_renewal_flag |
| Product term / period type | prior_bill_product_period_name |
| Product term / period quantity | prior_bill_product_period_qty |
| Expected receipts | expected_receipt_price_usd_amt |
| Renewal receipts | renewal_bill_receipt_price_usd_amt |
| Expiration / expiry / paid-through date | prior_bill_paid_through_mst_date |
| Billing date / bill date / billing due date | prior_bill_billing_due_mst_date |
| Renewal date | renewal_bill_modified_mst_date |
| Cancellation date | subscription_cancel_mst_date |

**Product term is two fields, not one.** `prior_bill_product_period_name` holds the unit as a string (`'year'`, `'month'`) and `prior_bill_product_period_qty` holds the numeric count of that unit. Filter by `period_name` alone to select a unit type (e.g., all monthly-billed terms: `prior_bill_product_period_name = 'month'`). Only add `period_qty` when the audience must be narrowed to a specific term length — e.g., `prior_bill_product_period_name = 'year' AND prior_bill_product_period_qty = 3` returns only 3-year terms. Do not invent a single combined "product_period" or "product_term" field — it does not exist on this table.

`historical_auto_renewal_flag` is the canonical auto-renew field for this table. Do not use `auto_renew_flag` — that column does not exist on `ba_commercial_success.eor_sizing_renewals` and will fail Helix SQL validation.

#### Trial conversion sizing fields

| Concept | Table / field |
|---|---|
| Sizing table | ba_commercial_success.eor_trial_360 |
| Product dimension lookup table | bigreporting.dim_product_snap |
| Trial product PNL line | prior_bill_product_pnl_line_name |
| Trial quantity (expiry equivalent) | trial_qty |
| Expected receipts (revenue potential) | expected_receipt_price_usd_amt |
| Actual paid revenue (GCR) | paid_gcr_usd_amt |
| Campaign anchor date / Expiration date | free_target_expiration_mst_date |
| Paid conversion date | paid_bill_mst_date |
| Trial cancellation date | subscription_cancel_mst_date |
| Trial type (BMAT/CMAT/FreeMAT/RMAT) | trial_type |
| Payment status (for UPM audiences) | payment_status_new |

#### Trial timing window definitions

Trial campaigns target users at specific points in their trial lifecycle. The trial_type field contains the trial window identifier.

| Trial Type | Description | Contact timing |
|---|---|---|
| BMAT | Before Mid-Trial | Early trial engagement |
| CMAT | Current Mid-Trial | Mid-trial engagement |
| FreeMAT | Free Mid-Trial | Late free trial |
| RMAT | Renewal Mid-Trial | Near trial expiration |

Filter by trial_type to select specific trial timing windows. These windows should be confirmed with the user during audience timing and eligibility (S1-S7) for trial workflows.

Example filter: `trial_type IN ('CMAT', 'RMAT')`

Do not use Helix sizing fields for setup-experiment or gather-results unless the user explicitly switches to opportunity sizing.

### Ratio metric definitions for power calculation

For sizing power calculations, both renewal rate and trial conversion rate are ratio metrics.

#### Renewal rate ratio metric

Default renewal ratio metric mapping:
- Numerator: renewal indicator or renewal units per eligible resource, derived from Helix baseline sizing data.
- Denominator: eligible expiring resource indicator or eligible resource count, derived from Helix baseline sizing data.
- Ratio metric value: numerator / denominator.

#### Trial conversion rate ratio metric

Default trial conversion ratio metric mapping:
- Numerator: paid conversion indicator (0/1) per eligible trial, derived from Helix baseline sizing data.
- Denominator: eligible trial start indicator (always 1 for eligible trials), derived from Helix baseline sizing data.
- Ratio metric value: numerator / denominator.

#### Moment calculation requirements (both renewal and trial)

When calculating ratio metric moments for power calculation, exclude zero values where instructed:
- numerator mean: mean of numerator values excluding 0
- numerator variance: variance of numerator values excluding 0; must be positive
- denominator mean: mean of denominator values excluding 0
- denominator variance: variance of denominator values excluding 0; must be positive
- covariance: covariance between numerator and denominator excluding zero-pair rows as instructed by the Hivemind ratio metric calculation requirements; must be non-zero when required by the calculator

If exact numerator/denominator row-level mapping is unclear, the owning Helix or power skill must ask the user to confirm the numerator and denominator definitions before power calculation.

### Confluence pages

**Source pages (reference only, fetch dynamically when needed):**
- Campaign documentation root: https://godaddy-corp.atlassian.net/wiki/spaces/REN/pages/110854749/Renewal+Channels
- Scorecard and metric mapping guide: https://godaddy-corp.atlassian.net/wiki/spaces/REN/pages/4073193958/Hivemind+Scorecard+Metric+Guide (Page ID: 4073193958)
- Jira custom field mappings: https://godaddy-corp.atlassian.net/wiki/spaces/REN/pages/4559275097 (Page ID: 4559275097)

**Opportunity sizing output location:**
- Parent page URL: https://godaddy-corp.atlassian.net/wiki/spaces/REN/pages/4515237230/Opportunity+Agent+Repo
- Parent page ID: 4515237230

### Jira ticket-creation defaults for intake workflow only

For ticket-creation (T1-T4), these defaults are authoritative:

| Concept | Default |
|---|---|
| Project | REN |
| Issue Type | Story |
| Platform | Hivemind |
| Value Type | Experiment |
| Business Unit | Corporate-Marketing |
| Functional Area | CM: Customer & Site |
| Business Alignment | Great 8 Corp |
| Test Objective | KPI Lift |
| Test Type | A/B/n |
| Hivemind Business Unit | Customer & Site |

Objective (`customfield_14600`, CM-only) does not exist on the REN project's Story screen. Do not set it.

Reporter/Assignee = blank for non-evergreen campaigns. For evergreen campaigns, set both to the mapped POC below.
Requested By = the user engaging with the agent, for ALL tickets. On REN this is `customfield_17008` ("Requestor"); on CM it is `customfield_18130` ("Requested By"). Use the field ID matching the target issue's project.

**Business Alignment (`customfield_23501`) note:** verified 2026-09-18 — REN's current option set (`I+M`, `Thrive`, `Start`, `Renewals`, `Brand Harvest`, `Specific Initiative`, `Other`) does not include `Great 8 Corp`. System admins are updating REN's field configuration to match CM's options; until that lands, if the write is rejected, disclose the failure rather than substituting a different value or blocking ticket creation on it.

**Evergreen campaign POC assignment:**
- Pre Expiry/Billing (page 3969122345): Kelley Fogarty (62cf64866eba7198372235ff)
- Post Expiry/Billing (page 3969122433): Myra P (712020:168fe71e-35e7-4c64-8ca6-f1c4e4780e4f)
- Other (page 3968565371): Andrew Masayestewa (62cf86c81e326fd93012f61b)

**Functional Requirements Template:** https://godaddy-corp.atlassian.net/wiki/spaces/REN/pages/3916315691

Do not derive these fixed defaults from Jira, Confluence, prior experiments, or user intake. Only allow override if the user explicitly asks to override ticket-creation defaults.

### Hivemind fixed setup-experiment defaults override

For setup-experiment, these defaults are authoritative:

| Concept | Default |
|---|---|
| Business unit | Customer & Site |
| Team name | Retention |
| Ad group | Renewals |
| Traffic type | shopper_id |
| Analysis type | frequentist |
| Test type | offsite |
| Category | Offsite |
| Alpha | 0.1 |
| Experiment classification | Seamless Experience: Site & Renewals |
| Experiment Insights | None |

Rules:
- Team name must be exactly Retention.
- Category must be exactly Offsite.
- Experiment Insights must be exactly None.
- Traffic type must be exactly shopper_id.
- Do not derive these fixed defaults from Jira, Confluence, prior experiments, dashboards, research, business analytics, or user intake.
- Do not ask the user to provide these values.
- Only allow override if the user explicitly asks to override setup defaults.

### Experiment power defaults for sizing workflow only

| Concept | Default |
|---|---|
| Confidence level | 95%, user confirmation required |
| Power | 80%, user confirmation required |
| Alpha scenario 1 | 0.1 |
| Alpha scenario 2 | 0.2 |

### Channel abbreviation mapping

| Channel | Abbreviation |
|---|---|
| Email | EM |
| SMS | SMS |
| WhatsApp | WA |

If multiple channels are present, use the primary channel only if clearly documented; otherwise mark REQUIRES USER CONFIRMATION.

### Channel consent field mapping (sizing only)

Source table: `ba_commercial_success.eor_sizing_renewals`.

| Channel | Consent field | Type | Opted-in value | Applied |
|---|---|---|---|---|
| SMS | sms_marketing_opted_in_flag | boolean | true | Always, when SMS is a selected channel |
| WhatsApp | whats_app_marketing_opted_in_flag | boolean | true | Always, when WhatsApp is a selected channel |
| Email | email_marketing_opted_in_flag | boolean | true | Conditionally — only when the communication is classified as non-transactional or research explicitly requires consent for Email. Default is transactional (no predicate). See "Email consent requirement (conditional)" in `eor-renewal-experiment-worfklow-audience-timing-and-eligibility.md`. |

These fields are genuine SQL boolean columns. Use the unquoted boolean literal `true` in predicates, not the string `'true'`.

`= true` correctly excludes both `false` and `NULL` (unrecorded consent), which is the required behavior for a consent filter. Do not rewrite this as `IS NOT FALSE` or otherwise include NULL rows.

Other channels not listed here do not have a consent predicate under this rule. Do not fabricate a consent field for a channel not listed in this table.

**Verified live column stats (2026-09-03), `ba_commercial_success.eor_sizing_renewals`:**
- sms_marketing_opted_in_flag: True 74,677,812 / False 205,670,971 / NULL 56,211,289
- whats_app_marketing_opted_in_flag: True 32,329,164 / False 30,167,377 / NULL 274,063,531
- email_marketing_opted_in_flag: True 209,803,776 / False 121,859,019 / NULL 4,897,277

### Campaign timing anchor rules for opportunity sizing only

Expiration-based campaigns use prior_bill_paid_through_mst_date.

Billing-based campaigns use prior_bill_billing_due_mst_date.

Absolute-send-date campaigns use the explicit send/eligibility/deployment/minimum_send date as send_eligibility_date.

Renewal availability for sizing:
renewal_bill_modified_mst_date IS NULL OR renewal_bill_modified_mst_date >= send_eligibility_date

Cancellation availability for sizing:
subscription_cancel_mst_date IS NULL OR subscription_cancel_mst_date >= send_eligibility_date

Availability predicates must be applied against send_eligibility_date, not directly against campaign_anchor_date.

### Scorecard and metric derivation fallback

For setup-experiment and sizing workflows, if Jira and user input do not explicitly provide scorecard or metric information, consult the scorecard and metric mapping page and recommend scorecard, decision metrics, and guardrail/supporting metrics for user approval before use.

### Helix conversation_id retry convention for sizing only

The master orchestrator must not invoke the Helix Conversation Session Manager.

The Helix-backed owning skill must invoke the Helix Conversation Session Manager immediately before the first actual Helix request if no valid helix_conversation_id exists.

If Helix fails with a timeout, transient session error, conversation error, context error, UUID/session/context error, or retryable failure during opportunity sizing or power baseline retrieval:
- Do not ask the user for a UUID.
- Reuse the existing validated helix_conversation_id.
- Retry, status check, or continuation must use the same conversation context.
- Do not generate a new UUID.
- Do not start a fresh Helix session.
- Do not retry with a different UUID.
- Use the same confirmed audience query contract.
