Audience, Timing, and Eligibility Rules
Scope
This skill applies only when workflow_intent = size_opportunity.

Do not use this skill for setup-experiment or gather-results workflows.

This skill owns sizing gates S1-S7.

Purpose
Derive and confirm audience eligibility rules, including product, market, timing anchor, send eligibility date, renewal/cancellation availability predicates, scorecard and decision metric assumptions, and a complete audience query contract.

Required inputs
This skill expects the master orchestrator to pass:

Jira ticket data.
Shared constants and field mappings.
Jira/Confluence research_context.
This skill must not call view_skill to load shared constants or research if sizing_shared_constants_loaded = true and sizing_research_context_loaded = true.

If the required dependency outputs are missing:

Return NEEDS_SIZING_DEPENDENCIES to the master orchestrator.
Do not load dependencies directly.
Do not inspect S8-S12 skills.
S1-S7 execution contract
This skill must:

Use shared constants and research_context as inputs.
Build an evidence-based audience and timing readout.
Produce the mandatory audience confirmation readout.
Ask the exact confirmation question in this skill.
Never call Helix.
Never invoke the Helix Conversation Session Manager.
Never open S8-S12 skills.
Never invoke view_skill for dependency skills after dependencies are loaded.
Block progression if any audience query contract row is missing, blocking, or unresolved.
Never render S1/S3/S4/S5/S6/S7 as visible sections, headings, or tables — see Display contract below. This applies unconditionally, not only when confidence is high.
Display contract (unconditional — no exceptions)
The S1, S3, S4, S5, S6, and S7 headings in this skill are internal processing steps for the agent to work through. They are never gate labels, headings, tables, or bullet breakdowns to render to the user, regardless of evidence quality, ambiguity, or confidence in the derivation.

During S1-S7, the only things visible to the user are:

The S2 date-range question (if not already confirmed).
The single compressed "Audience Package" readout (S2, after date range is confirmed).
A minimum-information clarifying question, only when S1-S7 is genuinely blocked per the Blocking behavior section below.
Do not print, under any framing ("Here's my research so far", "For transparency", "S1 — Campaign Identification", etc.):

A campaign identification writeup or table (S1 work).
A timing anchor validation writeup or table (S3 work).
A send eligibility derivation writeup (S4 work).
Availability predicate writeups (S5 work).
Scorecard/metric research writeups (S6 work).
The audience query contract table (S7 work) — this stays in the audit trail per the section below.
This holds even when confidence in the derivation is low. Low confidence in S1-S7 evidence is a reason to ask a targeted clarifying question (per Blocking behavior), not a reason to dump the full derivation to the user. If a future version adds a confidence-gated exception to this rule, it will be documented explicitly in this section — until then, treat this as absolute.

S1 - Campaign identification
Use research_context to derive campaign identification from:

Jira summary/title.
Jira description.
Jira comments.
Labels/components.
Linked Jira issues.
Linked Confluence pages.
If campaign identification is unavailable or ambiguous, ask only for the minimum campaign identification needed to continue.

Workflow type detection (renewal vs trial)
Before validating the timing anchor (S3), determine workflow_type = renewal or workflow_type = trial from research_context, Jira, and Confluence evidence gathered for S1.

Set workflow_type = trial when any of the following is explicitly present:

The audience is described as free trial, trial subscriptions, or free-to-paid (F2P) conversion.
A trial timing window code appears: BMAT, CMAT, FreeMAT, or RMAT (as a campaign name component, Jira label, or Confluence campaign identifier — e.g., "1250_BMAT-Failed-Billing", "CMAT F2P").
The audience is identified via the Free Trial identifier (subscription_status_id = 14) or the campaign documentation states the audience is trial non-canceled PNL details set to auto- or manual-renew.
The decision or guardrail metric is free_trial_conversion.
Campaign documentation places the campaign under a "FreeToPaid" or "Free to Paid Conversion" section (e.g., the Confluence campaign library grouping).
Set workflow_type = renewal when the audience is paid, non-canceled, non-trial subscriptions and none of the trial signals above are present. This remains the default when evidence is silent — do not set workflow_type = trial speculatively.

If evidence conflicts (e.g., campaign name suggests trial but decision metric suggests renewal), treat workflow_type as ambiguous and ask the user to confirm renewal vs trial before proceeding to S3. Do not guess.

State update:

workflow_type = renewal | trial
workflow_type_confirmed = true (or false if the user had to confirm ambiguity)
This determination governs which field mappings S3 and S5 use. It must complete before S3 runs — S3 must not default to renewal fields when workflow_type = trial.

Trial timing window confirmation (workflow_type = trial only)
When workflow_type = trial, also determine which specific trial timing window(s) the campaign targets: BMAT, CMAT, FreeMAT, or RMAT (see the trial timing window definitions in eor-renewal-experiment-worfklow-shared-constants-and-field-mappings.md). Derive this from the same evidence used to set workflow_type = trial — campaign name/ID, Jira labels, and Confluence campaign documentation typically state the window explicitly (e.g., "BMATPost", "1251_CMAT-Failed-Billing").

If more than one trial type is targeted by the same campaign, include all confirmed types.

If the trial window cannot be determined from available evidence, ask the user to confirm which of BMAT/CMAT/FreeMAT/RMAT applies before proceeding — do not guess or default to one arbitrarily.

State update:

trial_type_confirmed = ['BMAT'] (or the confirmed subset/superset of BMAT, CMAT, FreeMAT, RMAT)
trial_timing_window_confirmed = true
This flag is a required entry condition for eor-trial-experiment-workflow-helix-sizing-financial-impact.md (S9-S10). Do not advance to S8 with workflow_type = trial and trial_timing_window_confirmed unset.

UPM campaign detection
If campaign identification, Jira content, Confluence content, audience criteria, campaign name, communication name, linked documentation, or user confirmation explicitly references:

UPM
Update Payment
Update Payment Method
Update Payment Information
then:

payment_status_filter_required = true
Audience query contract behavior:

Include a payment-status audience requirement.
Classify the requirement as a Predicate.
Mark the requirement as requiring SQL translation during Helix sizing.
Do not infer UPM intent from:

Generic billing campaigns.
Renewal campaigns.
Expiration campaigns.
Payment-related language without explicit UPM or Update Payment terminology.
Collections language.
Failed-payment language.
Only set payment_status_filter_required = true when UPM or Update Payment terminology is explicitly present.

Channel consent requirement
If the confirmed selected channel(s) for the campaign include SMS or WhatsApp:

sms_consent_filter_required = true when SMS is a selected channel.
whatsapp_consent_filter_required = true when WhatsApp is a selected channel.
Audience query contract behavior:

For each channel with a consent filter required, include a consent audience requirement.
Classify the requirement as a Consent predicate.
Mark the requirement as requiring SQL translation during Helix sizing.
Rules:

Derive selected channel(s) from Jira customfield_24501 / research_context, the same source used for the audience confirmation readout {channel} value. Do not ask the user for channel when it is already confirmed there.
Apply the consent filter only for the channel(s) actually selected for the campaign. Do not apply SMS consent to a WhatsApp-only campaign or vice versa.
If both SMS and WhatsApp are selected channels, both consent predicates are required (combined with AND).
Email and other channels do not require a consent predicate under this rule.
Do not infer consent requirements for a channel that is not an explicitly selected channel.
Do not fabricate a consent field for a channel not covered by the mapping in eor-renewal-experiment-worfklow-shared-constants-and-field-mappings.md. If an unmapped channel requires consent, stop and ask for the canonical field name before including it in the contract.
Email consent requirement (conditional)
Unlike SMS and WhatsApp, Email consent is not required by default. Most renewal/expiration communications sent by Email are transactional by nature of the content being sent, so the default assumption is email_consent_filter_required = false.

Set email_consent_filter_required = true only when Email is a selected channel AND at least one of the following is true:

research_context's consent/communication-classification evidence (consent_evidence, or Jira/Confluence content) explicitly classifies the communication as non-transactional — e.g., Marketing, Promotional, Newsletter, or similar.
research_context, Jira, or Confluence explicitly states that consent or opt-in is required for the Email channel on this campaign.
Rules:

Derive this from the same research_context evidence already gathered for S1 (consent_evidence in the research skill's available_context). Do not perform new retrieval solely for this check.
Do not ask the user upfront whether the Email communication is transactional. Only surface it if the evidence is genuinely ambiguous or conflicting, and only as part of the minimum missing information needed to unblock S1-S7 — do not treat it as a blocking stop on its own.
If evidence is silent (no classification found either way), keep the default: email_consent_filter_required = false. Silence is not evidence of a marketing classification.
Do not infer non-transactional status from the campaign being a renewal/win-back/upsell effort alone — transactional renewal and billing communications on Email are still transactional by default under this rule.
Record the basis for the decision (evidence found, or default applied) for audit, even though the audience confirmation readout stays compressed.
HVC (High Value Customer) targeting requirement
If campaign identification, Jira content, Confluence content, audience criteria, campaign name, communication name, linked documentation, or user confirmation explicitly references:

HVC
High Value Customer
High Value Customers
High-Value Customer
then:

hvc_filter_required = true
Audience query contract behavior:

Include an HVC audience requirement.
Classify the requirement as a Predicate.
Mark the requirement as requiring SQL translation during Helix sizing.
The Helix sizing query will join to customer360.customer_life_cycle_vw on shopper_id and partition_eval_mst_date matching the prior_bill_paid_through_mst_date, filtering for high_value_customer_flag = true.
Do not infer HVC intent from:

Generic high-spend or premium customer language.
VIP customer references without explicit HVC terminology.
Revenue-tier language without explicit HVC or High Value Customer terminology.
Only set hvc_filter_required = true when HVC or High Value Customer terminology is explicitly present.

S2 - Date range preference and audience confirmation
Date range for opportunity sizing
Before displaying the audience confirmation, ask the user for their preferred date range for opportunity sizing:

Question format:
"What date range would you like to use for opportunity sizing?

12mo (last 12 months)
6mo (last 6 months)
3mo (last 3 months) [DEFAULT]
1mo (last month)"
Rules:

If user doesn't specify, default to 3mo
Accept variations: "12 months", "6 months", "3 months", "1 month", "12", "6", "3", "1"
Store as opportunity_sizing_date_range_months with values: 12, 6, 3, or 1
This date range will be applied in the Helix sizing query to filter campaign_anchor_date
State update:

opportunity_sizing_date_range_months = {user selection or 3}
opportunity_sizing_date_range_confirmed = true
Audience confirmation
After date range is confirmed, display the compressed audience confirmation readout and ask for confirmation.

S3 - Campaign timing anchor validation
Before audience confirmation, validate timing anchor including:

Campaign date anchor type.
Anchor field.
Timing type.
Offset.
Derived send eligibility expression.
Renewal/cancellation fields.
Inclusive availability predicates.
Campaign timing anchor types depend on workflow_type (from S1). workflow_type must be resolved before selecting an anchor type — never default to a renewal anchor field when workflow_type = trial.

workflow_type = renewal:

Expiration-based campaigns use prior_bill_paid_through_mst_date.
Billing-based campaigns use prior_bill_billing_due_mst_date.
Absolute-send-date campaigns use the explicit send/eligibility/deployment/minimum_send date as send_eligibility_date.
workflow_type = trial:

All trial timing windows (BMAT, CMAT, FreeMAT, RMAT) use free_target_expiration_mst_date as campaign_anchor_date. There is no expiration-based vs billing-based distinction for trials — trials bill on their target expiration date, so this single field covers both pre- and post-billing trial sends.
Absolute-send-date campaigns use the explicit send/eligibility/deployment/minimum_send date as send_eligibility_date, same as renewal.
Do not use prior_bill_paid_through_mst_date or prior_bill_billing_due_mst_date for a trial campaign — those fields belong to ba_commercial_success.eor_sizing_renewals, not ba_commercial_success.eor_trial_360. See eor-renewal-experiment-worfklow-shared-constants-and-field-mappings.md (Trial conversion sizing fields) and FIELD_MAPPING_RENEWAL_VS_TRIAL.md for the full field-by-field mapping.
If the campaign's timing language (e.g., "days after bill_date", "Sends N days post-expiration") sounds like it matches a renewal anchor description, still confirm workflow_type from S1 first — trial campaigns are frequently documented using the same "post-billing" / "post-expiration" language as renewal campaigns, but resolve to a different underlying field.

S4 - Send eligibility date derivation
Derive and confirm send_eligibility_date from the validated timing anchor.

If offset, direction, or anchor field is ambiguous, ask only for the missing timing detail.

S5 - Renewal/conversion and cancellation availability predicates
Validate availability predicates against send_eligibility_date. The success-event field depends on workflow_type (from S1); the cancellation field is the same for both.

workflow_type = renewal — renewal availability predicate:
renewal_bill_modified_mst_date IS NULL OR renewal_bill_modified_mst_date >= send_eligibility_date

workflow_type = trial — conversion availability predicate:
paid_bill_mst_date IS NULL OR paid_bill_mst_date >= send_eligibility_date

Cancellation availability predicate (same field for both workflow_type):
subscription_cancel_mst_date IS NULL OR subscription_cancel_mst_date >= send_eligibility_date

Stop if availability is applied directly to the anchor date instead of send_eligibility_date. Stop if the trial workflow uses renewal_bill_modified_mst_date, or the renewal workflow uses paid_bill_mst_date — these are not interchangeable across the two sizing tables.

S6 - Scorecard and decision metrics
Confirm scorecard and decision metrics from Jira, research_context, scorecard mapping, or user confirmation.

If scorecard or metric information is unavailable, disclose the missing information and mark it as unavailable or required input as appropriate.

Compressed audience confirmation readout
Display the audience package in compressed format:

Audience Package ({jira_ticket}):

Products: {product_list_comma_separated}
Market: {market} | Channel: {channel} | Auto-renew: {auto_renew_label}
Timing: {timing_summary_one_line}
Scorecard: {scorecard_name} | Decision: {decision_metric}

Confirm?
Where:

{jira_ticket} = Jira ticket key
{product_list_comma_separated} = Product PNL hierarchy or line, comma-separated
{market} = Market(s), comma-separated if multiple
{channel} = Channel(s), comma-separated if multiple
{auto_renew_label} = Auto-renew status (e.g., "On", "Off", "All")
{timing_summary_one_line} = Timing anchor field + operator + offset in one line (e.g., "prior_bill_paid_through_mst_date >= current_date - 14 days")
{scorecard_name} = Scorecard name or "Not specified"
{decision_metric} = Decision metric or "Not specified"
Audit trail (do not display to user)
Maintain but do not display:

Full inclusion filter predicates
Full exclusion filter predicates
Product exclusions
Communication classification
Consent predicates
UPM detection flag (payment_status_filter_required)
Complete availability predicates
All assumptions
These details are required for audience query contract generation but should not be shown in the confirmation readout.

This audit-only rule is unconditional — see the Display contract section above. It does not have a confidence-based exception: low confidence in the underlying evidence is handled by asking a targeted clarifying question, never by displaying the audit trail.

S7 - Audience query contract
After confirmation, map every confirmed row to include one of:

Predicate.
Include predicate.
Exclude predicate.
Availability predicate.
Consent predicate.
No-filter/default.
Blocking unresolved mapping.
If payment_status_filter_required = true, include an audience query contract row:

Classification: Predicate
Description: Current payment status eligibility required for UPM audience.
Helix SQL translation required: Yes
If sms_consent_filter_required = true, include an audience query contract row:

Classification: Consent predicate
Description: SMS marketing consent required for SMS channel audience.
Helix SQL translation required: Yes
If whatsapp_consent_filter_required = true, include an audience query contract row:

Classification: Consent predicate
Description: WhatsApp marketing consent required for WhatsApp channel audience.
Helix SQL translation required: Yes
If email_consent_filter_required = true, include an audience query contract row:

Classification: Consent predicate
Description: Email marketing consent required — communication classified as non-transactional or consent explicitly required per research.
Helix SQL translation required: Yes
If Email is a selected channel and email_consent_filter_required = false (the default), do not include a consent predicate row for Email. Optionally note in the audit trail that Email was treated as transactional by default.

If hvc_filter_required = true, include an audience query contract row:

Classification: Predicate
Description: High Value Customer (HVC) targeting required — audience limited to customers with high_value_customer_flag = true.
Helix SQL translation required: Yes
Always include an explicit sizing-table row derived from workflow_type:

workflow_type = renewal: Sizing table = ba_commercial_success.eor_sizing_renewals.
workflow_type = trial: Sizing table = ba_commercial_success.eor_trial_360, plus a trial-type include predicate (trial_type IN (...)) for the confirmed BMAT/CMAT/FreeMAT/RMAT window(s).
Do not proceed to Helix sizing if any confirmed criterion is missing, blocking, or unresolved.

Completion contract
When S1-S7 are complete, set:

workflow_type = renewal | trial (resolved in S1, must be set before this contract completes)
trial_timing_window_confirmed = true and trial_type_confirmed = [...] (only when workflow_type = trial)
audience_confirmed = true
audience_query_contract_complete = true
campaign_timing_anchor_validated = true
send_date_operator_validation_complete = true
scorecard_metrics_confirmed = true
sizing_s1_s7_complete = true
current_gate = S8
Then return control to the master orchestrator route resolver.

Blocking behavior
Stop only when:

Required campaign identification cannot be derived.
workflow_type (renewal vs trial) cannot be determined and evidence is genuinely conflicting or silent in a way that makes the default unsafe.
workflow_type = trial and the specific trial timing window (BMAT/CMAT/FreeMAT/RMAT) cannot be determined.
Timing anchor is ambiguous.
Anchor field is wrong or unresolved.
Send eligibility date cannot be derived.
Availability predicates cannot be safely constructed.
Audience confirmation is missing.
Audience query contract contains unresolved, missing, or blocking rows.
When blocked, ask only for the minimum missing information needed to unblock S1-S7.
