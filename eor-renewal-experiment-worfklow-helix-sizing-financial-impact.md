Helix Sizing and Financial Impact
Scope
This skill applies only when workflow_intent = size_opportunity and current_gate is S9 or S10.

Do not use this skill for setup-experiment or gather-results workflows.

Purpose
Run baseline sizing using Helix and calculate renewal opportunity impact using the confirmed audience query contract.

Verbosity requirement - Silent execution
Do not provide intermediate reasoning, exploration updates, progress messages, query-planning commentary, or status logs.

Perform all analysis silently and return only:

Final SQL query
Final baseline metrics
Validation findings (if any)
Final recommendations
Do not stream or narrate:

Table exploration steps
Join evaluation reasoning
Field discovery commentary
Query construction thought process
Status updates during execution
Rationale:
Streaming status updates are repeatedly injected into the companion agent's context, consuming tokens without adding business value. All intermediate work must be performed silently.

Context caching requirement
Store retrieved campaign context as session state. Do not repeat previously confirmed campaign metadata unless it changes or impacts calculations.

Do not re-print in every response:

Campaign ID
Jira ticket details
Confluence context
Audience query contract (unless validating changes)
Previously confirmed timing windows
Previously confirmed channel requirements
Only reference session state when:

Validating a change to confirmed context
Detecting an inconsistency that requires user attention
Displaying final results that depend on specific context values
Entry requirements
Before this skill runs:

sizing_s1_s7_complete = true
lift_confirmed = true
audience_query_contract_complete = true
opportunity_sizing_date_range_confirmed = true
opportunity_sizing_date_range_months is set (1, 3, 6, or 12)
current_gate is S9 or S10
If any entry requirement is missing, stop and return control to the master orchestrator with the missing requirement.

Helix Conversation Session Manager ownership
This skill, not the master orchestrator, owns Helix Conversation Session Manager invocation for S9-S10.

Immediately before the first actual Helix request, this skill must check whether a valid helix_conversation_id exists.

If no valid helix_conversation_id exists, invoke eor-renewal-experiment-worfklow-helix-conversation-session.md.

This skill must not generate, predict, infer, type, approximate, fabricate, or manually compose conversation_id.

Every Helix request payload must include:

{
  "conversation_id": "{helix_conversation_id}",
  "agent": "analysis",
  "instructions": "Run the Analysis Agent to answer the SQL query. Use the SQL provided below. Do not perform schema discovery. Do not search for alternate tables. Do not validate against unrelated datasets. Execute the query and return results only."
}
Helix Analysis Agent routing:
All Helix queries must explicitly route to the Analysis Agent with enhanced instructions that disable schema discovery and validation cycles. The Helix team has confirmed that explicit routing with validated SQL reduces execution time from ~5 minutes to ~2.5 minutes by skipping the discovery/validation cycle.

Reuse the same conversation_id for all Helix sizing calls, retries, follow-up turns, timeout recovery, status checks, and continuation requests in the same opportunity sizing workflow.

Helix Status Checker integration
When submitting the Helix baseline sizing query (which may take 7-10 minutes via Helix MCP):

Submit Helix SQL query via MCP
Immediately invoke: eor-renewal-experiment-worfklow-helix-status-poll.md
Pass state: {helix_conversation_id, max_wait_seconds: 600}
Status checker will poll query status every 20 seconds
When status checker returns status="completed", process results
If status checker returns status="error" or "timeout", handle per existing exception logic
Do NOT reload this full skill's prompt during Helix status checks.
The status checker is a lightweight 5K skill that handles all polling.

Rationale: Helix queries on ba_commercial_success.eor_sizing_renewals can take 4-5 minutes in production.
With v2.2.2 optimizations:

20-second status polling (vs 5-second in v2.2.1)
Status checker: 5K prompt × 15 checks = 75K tokens (vs 885K with full prompt reloads)
S11 no longer runs Helix query (agent-side math instead)
Saves: ~810K tokens per sizing + eliminates 4-minute S11 Helix query
Mandatory audience query contract requirement
Build the Helix query only from the confirmed audience query contract. Do not run from a narrative summary alone.

UPM audience SQL construction requirement
If the confirmed audience query contract indicates:

payment_status_filter_required = true
then the Helix sizing query must include the following audience predicate:

payment_status_new = &#39;Not Billable&#39;
Rules:

Use payment_status_new as the canonical payment-status field.
Use the current payment status value from the sizing source table.
Do not infer payment status from renewal status.
Do not infer payment status from cancellation status.
Do not substitute another payment-related field.
Do not ask the user for a payment-status value when a UPM audience has already been identified.
Always apply the predicate exactly as:
payment_status_new = &#39;Not Billable&#39;
when payment_status_filter_required = true.

Validation requirements:

Validate that the audience query contract explicitly contains the UPM-derived payment status requirement.
Display that the payment-status filter has been applied before query execution.
Preserve all other audience predicates from the audience query contract.
Execution display:

UPM audience detected: Yes
Payment status filter applied: payment_status_new = 'Not Billable'
Channel consent SQL construction requirement
If the confirmed audience query contract indicates:

sms_consent_filter_required = true
then the Helix sizing query must include:

sms_marketing_opted_in_flag = true
If the confirmed audience query contract indicates:

whatsapp_consent_filter_required = true
then the Helix sizing query must include:

whats_app_marketing_opted_in_flag = true
If the confirmed audience query contract indicates:

email_consent_filter_required = true
then the Helix sizing query must include:

email_marketing_opted_in_flag = true
Email's consent predicate is conditional (only required when the audience query contract explicitly sets email_consent_filter_required = true — see the Email consent requirement rule in eor-renewal-experiment-worfklow-audience-timing-and-eligibility.md). Do not apply it by default for Email campaigns.

If multiple channel consent requirements are set (any combination of SMS, WhatsApp, Email), include all applicable predicates combined with AND.

Rules:

Use the exact field names and canonical field mapping in eor-renewal-experiment-worfklow-shared-constants-and-field-mappings.md.
These are boolean columns. Use the unquoted boolean literal true, not the string 'true'.
Do not apply a channel's consent predicate unless the audience query contract explicitly requires it for that channel.
Do not substitute a different opted-in field for SMS, WhatsApp, or Email.
Do not infer consent requirements from campaign language alone; only apply when the audience query contract row is present (derived from confirmed selected channels and, for Email, the conditional classification check in S1).
Validation requirements:

Validate that the audience query contract explicitly contains each required channel consent predicate before query execution.
Display each applied consent filter before query execution.
Preserve all other audience predicates from the audience query contract.
Execution display:

Channel(s) requiring consent filter: {channel_list}
Consent filter(s) applied: {predicate_list}
Campaign timing anchor SQL construction requirement
The query must explicitly derive campaign_anchor_date and send_eligibility_date before applying renewal/cancellation availability logic.

Expiration-based example:
prior_bill_paid_through_mst_date AS campaign_anchor_date,
prior_bill_paid_through_mst_date + INTERVAL '18 days' AS send_eligibility_date

Billing-based example:
prior_bill_billing_due_mst_date AS campaign_anchor_date,
prior_bill_billing_due_mst_date - INTERVAL 'X days' AS send_eligibility_date

Availability predicates:
renewal_bill_modified_mst_date IS NULL OR renewal_bill_modified_mst_date >= send_eligibility_date
subscription_cancel_mst_date IS NULL OR subscription_cancel_mst_date >= send_eligibility_date

Date range filter requirement
The query must apply a date range filter based on the user's confirmed opportunity_sizing_date_range_months value from S2.

Required filter:

campaign_anchor_date >= CURRENT_DATE - INTERVAL &#39;{opportunity_sizing_date_range_months} months&#39;
AND campaign_anchor_date < CURRENT_DATE
Historical date validation (CRITICAL):
All date filters must reference historical activity only. The agent has previously failed by including future-dated expiration or bill dates in sizing results.

Campaign anchor dates (renewal_bill_billing_due_mst_date) must be < CURRENT_DATE
Availability event dates (renewal_bill_modified_mst_date, subscription_cancel_mst_date) must be in the past when present
Date range upper bound must not exceed today
Blocking behavior:
If any date predicate could match future-dated records, stop and return validation error before query execution.

Validation:

Verify opportunity_sizing_date_range_months is set (should be 1, 3, 6, or 12)
Default to 3 if somehow missing
Apply the filter to campaign_anchor_date (not send_eligibility_date)
Validate that upper bound excludes future dates
Display the applied date range filter before query execution
Example display:

Date range filter: Last {opportunity_sizing_date_range_months} months (historical only)
Filter SQL: campaign_anchor_date >= CURRENT_DATE - INTERVAL &#39;{opportunity_sizing_date_range_months} months&#39;
            AND campaign_anchor_date < CURRENT_DATE
Validate correct anchor, send eligibility derivation, inclusive predicates, date range filter, historical date constraint, complete audience contract, and any required UPM payment-status predicate before execution.

Query outputs
Return distinct shoppers, expiry quantity, renewal quantity, expected receipts, renewal receipts, baseline renewal rate, average receipt per renewed unit, baseline yield per expiring unit, average monthly shoppers, average daily shoppers, and ratio-metric moments required for power calculation.

Output display format
Display baseline sizing results in compact format:

Baseline Sizing ({jira_ticket}):

Shoppers: {total} ({daily}/day) | Pop: {annual}
Baseline rate: {rate}% | Revenue: ${revenue}

[Detailed breakdown in audit only]
Move verbose table displays and row-level details to audit section only.

Renewal rate ratio metric moment outputs for power calculation
Helix sizing must provide or enable calculation of:

Numerator mean excluding 0.
Numerator variance excluding 0; must be positive.
Denominator mean excluding 0.
Denominator variance excluding 0; must be positive.
Covariance excluding 0 as required; must be non-zero when required by the calculator.
If moment calculations require unclear row-level numerator and denominator definitions, stop and ask the user to confirm the numerator/denominator mapping.

Anomalous result handling
If, during baseline sizing execution, the agent identifies a result that is implausible, inconsistent with the confirmed audience query contract, or otherwise prompts the agent to re-run the query as a sanity check (an "anomalous result"):

Set sizing_anomaly_detected = true.
Record sizing_anomaly_description: what specifically looked wrong (e.g., implausible baseline rate, unexpected row counts, values inconsistent with the audience query contract).
Record sizing_anomaly_action_taken: what the agent did in response (e.g., "re-ran query with same audience query contract").
Do not silently clear sizing_anomaly_detected because a retry produced a result that looks more plausible. A retry succeeding is not the same as the user confirming the result is correct.
Do not set baseline_sizing_complete = true while sizing_anomaly_detected = true and sizing_anomaly_user_confirmed is not true.
This flag and its stop behavior are enforced by eor-renewal-experiment-worfklow-exception-handling-and-audit.md. This skill's only responsibility is to set the flag accurately and disclose it — do not add separate blocking logic here beyond what completion contract already requires.

When sizing_anomaly_detected = true, disclosing it and asking for confirmation must be the last thing this skill does before ending its turn. Do not continue into S11 power calculation in the same response, even if the retried query looks clean. Return control to the master orchestrator with current_gate still at S10 and wait for the user's next message before anything in S11 begins.

Completion contract
When baseline sizing and ratio moments are complete:

baseline_sizing_complete = true
ratio_metric_inputs_complete = true
sizing_s9_s10_complete = true
current_gate = S11
If sizing_anomaly_detected = true, baseline_sizing_complete must remain false until sizing_anomaly_user_confirmed = true, even if the retried query otherwise looks complete. Return control to the master orchestrator route resolver with the anomaly disclosed rather than advancing current_gate.

Return control to the master orchestrator route resolver.

Retry and timeout behavior
If Helix returns a timeout, transient session error, context error, conversation error, UUID/session/context error, or retryable failure:

Reuse the existing validated helix_conversation_id.
Do not generate a new UUID.
Do not start a fresh Helix session.
Retry, status check, or continuation must use the same conversation context.
Use the same confirmed audience query contract.
