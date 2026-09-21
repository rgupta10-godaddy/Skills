## Jira and Confluence Research

### Purpose

Retrieve Jira and Confluence context needed by the selected workflow without pulling in unrelated workflow validation.

This skill is dependency-only. It returns research context only and then exits.

### Context caching requirement

**Store retrieved campaign context as session state. Do not repeat previously confirmed campaign metadata unless it changes or impacts calculations.**

After initial retrieval, do not re-print in subsequent workflow responses:
- Campaign ID
- Jira ticket summary and description
- Jira custom field values (hypothesis, channels, cohorts, Hivemind link)
- Confluence document links and summaries
- Labels, components, linked issues

Only re-reference session state when:
- A validation requires checking against specific context values
- An inconsistency is detected that requires user attention
- Final results display depends on specific context fields

**Rationale:**
Campaign context (campaign IDs, labels, Jira details, audience definitions, Confluence context) is repeatedly printed in every workflow response, consuming tokens without adding value. After initial retrieval and confirmation, this context should be stored as session state and referenced only when necessary.

### Dependency-only execution contract

This skill must not:
- Invoke `view_skill`.
- Open the shared constants skill.
- Open another workflow skill.
- Route to another gate.
- Ask the user for missing workflow fields before returning research context.
- Perform Helix sizing.
- Perform lift validation.
- Perform experiment power calculation.
- Create Hivemind experiments.
- Prepare Jira writebacks.

This skill must:
- Use the selected workflow intent to limit retrieval scope.
- Use already-retrieved Jira issue data when available.
- Retrieve only additional linked Jira or Confluence context required by the selected workflow.
- Return a structured research_context object to the caller.
- Set the relevant retrieval state flags.
- Exit immediately after returning research_context.

### Workflow-aware retrieval

Always respect workflow_intent.

If workflow_intent = size_opportunity, retrieve campaign, audience, timing, product, market, eligibility, scorecard, metric, and hypothesis context needed by S1-S7.

If workflow_intent = setup_experiment, retrieve only fields needed to prepare a Hivemind experiment scorecard preview. Do not validate sizing, lift, Helix, power, audience query contract, or Confluence publishing.

If workflow_intent = gather_results, retrieve only fields needed to parse Hivemind link and prepare results updates. Do not validate sizing, setup scorecard creation, lift, Helix, power, or Confluence publishing.

### Common Jira fields to retrieve when available

- Summary / title
- Description
- Comments
- Labels/components/linked issues
- Hypothesis: customfield_24203
- Selected channels: customfield_24501
- Number of variants / cohorts / segments targeted: customfield_17203
- Hivemind link: customfield_34621
- End date: customfield_14200
- Outcome: customfield_17001
- Outcome details: customfield_17006

### Size-opportunity retrieval contract

For workflow_intent = size_opportunity, retrieve or summarize these fields when available:
- Jira ticket key and URL.
- Summary / title.
- Description.
- Comments.
- Labels/components.
- Linked Jira issues.
- Linked Confluence pages.
- Campaign or communication name.
- Product or product PNL context.
- Product exclusions.
- Market: customfield_16518.
- Channel / selected channels: customfield_24501.
- Audience or eligibility criteria.
- Auto-renew criteria.
- Consent or communication classification.
- Timing anchor language, including expiration, billing, paid-through date, bill due date, send date, deployment date, offset, and timing direction.
- Renewal and cancellation availability logic if documented.
- Scorecard and decision metrics if documented.
- Hypothesis and lift assumptions if documented.

For S1-S7, retrieve raw evidence and concise summaries. Do not attempt to complete S1-S7 gates inside this research skill.

Do not derive the final audience query contract. That belongs to eor-renewal-experiment-worfklow-audience-timing-and-eligibility.md.

Do not stop because some sizing context is missing. Return the available context and mark missing areas.

### Size-opportunity Confluence completion contract

For workflow_intent = size_opportunity, S1-S7 research is not complete until Confluence lookup has been attempted and explicitly recorded.

Before setting sizing_research_context_loaded = true, this skill must complete the following checks in order:

1. Jira context checked
   - Use the already-retrieved Jira ticket when available.
   - Inspect Jira summary/title, description, comments, labels/components, known custom fields, and linked issues when available.

2. Linked Jira context checked when present
   - If Jira issue links exist, inspect linked Jira issues that are relevant to campaign, audience, timing, product, market, eligibility, scorecard, metrics, hypothesis, lift, prior test context, or campaign documentation.
   - If no linked Jira issues exist, set linked_jira_context_checked = false and include linked_jira_context_status = NO_LINKED_JIRA_FOUND in research_context.

3. Linked Confluence context checked when present
   - If the Jira ticket contains linked Confluence pages, remote links, wiki links, pasted Confluence URLs, or application links, retrieve and inspect those pages.
   - Extract campaign, audience, timing, product, market, eligibility, scorecard, metric, and hypothesis evidence from linked Confluence pages.
   - If linked Confluence pages exist but cannot be accessed, set linked_confluence_context_checked = false, set linked_confluence_context_status = ACCESS_ERROR, include the error in retrieval_warnings, and continue with Jira plus shared-library context when safe.
   - If no linked Confluence pages exist, set linked_confluence_context_checked = false and linked_confluence_context_status = NO_LINKED_CONFLUENCE_FOUND.

4. Shared Confluence campaign documentation library checked
   - Always check the shared Confluence campaign documentation root for size_opportunity unless Confluence access is unavailable.
   - Use the campaign documentation root from shared constants: https://godaddy-corp.atlassian.net/wiki/x/XYKbBg.
   - Search or inspect the shared campaign documentation library using available Jira-derived identifiers, including summary/title terms, campaign or communication names, campaign numbers, product terms, market terms, channel terms, and linked-document titles.
   - Extract any relevant campaign, audience, timing, product, market, eligibility, scorecard, metric, and hypothesis evidence.
   - Set shared_confluence_campaign_library_checked = true when this lookup is attempted, even if no matching page is found.
   - If no matching campaign documentation is found, set shared_confluence_campaign_library_status = NOT_FOUND and continue.
   - If Confluence access fails, set shared_confluence_campaign_library_checked = false, set shared_confluence_campaign_library_status = ACCESS_ERROR, include the error in retrieval_warnings, and continue with Jira-derived evidence only if safe.

5. Shared scorecard and metric mapping library checked when metric context is missing or incomplete
   - If scorecard template, decision metrics, guardrail metrics, or supporting metrics are missing or ambiguous after Jira, linked Confluence, and campaign documentation lookup, check the shared scorecard and metric mapping page.
   - Use the scorecard and metric mapping page from shared constants: https://godaddy-corp.atlassian.net/wiki/x/5gHI8g.
   - Set shared_confluence_scorecard_mapping_checked = true when this lookup is attempted.
   - If scorecard or metric mapping is not needed because Jira or campaign documentation provides complete metric context, set shared_confluence_scorecard_mapping_checked = false and shared_confluence_scorecard_mapping_status = NOT_NEEDED.
   - If no mapping is found, set shared_confluence_scorecard_mapping_status = NOT_FOUND and continue with missing or ambiguous metric context recorded.
   - If access fails, set shared_confluence_scorecard_mapping_checked = false, set shared_confluence_scorecard_mapping_status = ACCESS_ERROR, and include the error in retrieval_warnings.

### Size-opportunity Confluence completion gate

For workflow_intent = size_opportunity, do not set sizing_research_context_loaded = true until these fields have explicit true/false or status values in research_context:

- jira_retrieval_complete
- linked_jira_context_checked
- linked_jira_context_status
- linked_confluence_context_checked
- linked_confluence_context_status
- shared_confluence_campaign_library_checked
- shared_confluence_campaign_library_status
- shared_confluence_scorecard_mapping_checked
- shared_confluence_scorecard_mapping_status

If the shared campaign documentation library is skipped, the skip must be explicit in retrieval_warnings and shared_confluence_campaign_library_status must be one of:
- ACCESS_ERROR
- NOT_AVAILABLE
- SKIPPED_USER_APPROVED

Do not silently skip the shared Confluence campaign documentation library.

If shared Confluence campaign library lookup fails because of access or tool failure, do not loop or retry indefinitely. Return the failure status in research_context and allow the S1-S7 owning skill to decide whether Jira-only context is sufficient or whether to ask the user for missing details.

### Required research_context output

Return this structure to the caller:

```json
{
  "workflow_intent": "size_opportunity | setup_experiment | gather_results",
  "jira_ticket": "...",
  "jira_retrieval_complete": true,
  "linked_jira_context_checked": true_or_false,
  "linked_jira_context_status": "CHECKED | NO_LINKED_JIRA_FOUND | ACCESS_ERROR | NOT_APPLICABLE",
  "linked_confluence_context_checked": true_or_false,
  "linked_confluence_context_status": "CHECKED | NO_LINKED_CONFLUENCE_FOUND | ACCESS_ERROR | NOT_APPLICABLE",
  "shared_confluence_library_checked": true_or_false,
  "shared_confluence_campaign_library_checked": true_or_false,
  "shared_confluence_campaign_library_status": "CHECKED | NOT_FOUND | ACCESS_ERROR | NOT_AVAILABLE | SKIPPED_USER_APPROVED | NOT_APPLICABLE",
  "shared_confluence_scorecard_mapping_checked": true_or_false,
  "shared_confluence_scorecard_mapping_status": "CHECKED | NOT_FOUND | ACCESS_ERROR | NOT_AVAILABLE | NOT_NEEDED | NOT_APPLICABLE",
  "sources_checked": [
    "jira_fields",
    "jira_summary",
    "jira_description",
    "jira_comments",
    "labels_components",
    "linked_jira",
    "linked_confluence",
    "shared_confluence_campaign_library",
    "shared_confluence_scorecard_mapping"
  ],
  "available_context": {},
  "missing_context": [],
  "ambiguities": [],
  "retrieval_warnings": [],
  "blocking_retrieval_errors": []
}
```

For size_opportunity, available_context should include these keys when available:
- campaign_identification_evidence
- campaign_documentation_evidence
- audience_evidence
- product_evidence
- market_evidence
- channel_evidence
- timing_anchor_evidence
- send_offset_evidence
- renewal_availability_evidence
- cancellation_availability_evidence
- consent_evidence
- scorecard_metric_evidence
- hypothesis_lift_evidence

Each evidence entry should include:
- value
- source_type: Jira field, Jira summary, Jira description, Jira comment, linked Jira, linked Confluence, shared campaign library, shared scorecard mapping
- source_name
- source_url when available
- confidence: High, Medium, or Low
- notes

### Sizing handoff behavior

When workflow_intent = size_opportunity:
- Set sizing_research_context_loaded = true only after the Confluence completion gate is satisfied.
- Preserve current_gate.
- If current_gate is null, the master orchestrator must set current_gate = S1.
- Return research_context to the orchestrator or current owning skill.
- Do not route to S8-S12.
- Do not reopen the shared constants skill.
- Do not ask what to do next.

### Setup-experiment retrieval

For setup workflow retrieve:
- Ticket Summary / Title
- Hypothesis
- Experiment Theme
- Selected Channels
- Calculator ID if present
- Scorecard template if present
- Decision metrics
- Guardrail metrics
- Hivemind Parent ID if present

If scorecard or metric fields are missing, consult Confluence mapping only through the retrieval tools available to this skill and return findings to the setup scorecard builder. Do not ask the user before preview.

The size-opportunity Confluence completion contract does not force shared campaign documentation lookup for setup_experiment unless the setup scorecard builder needs it to derive setup fields.

### Gather-results retrieval

For gather-results workflow retrieve:
- Hivemind link: customfield_34621
- Current end date: customfield_14200
- Current outcome: customfield_17001
- Current outcome details: customfield_17006
- Current IGCR values if accessible: customfield_26621, customfield_26620

The size-opportunity Confluence completion contract does not apply to gather_results.

### Failure handling

If Jira retrieval fails, return a blocking_retrieval_errors entry and do not fabricate context.

If linked Confluence retrieval is unavailable, return Jira-derived context plus shared campaign library findings when possible, mark linked_confluence_context_checked = false, and include the retrieval error in retrieval_warnings unless safe continuation is impossible.

If shared Confluence campaign library retrieval is unavailable, do not loop. Return Jira and linked-context findings, set shared_confluence_campaign_library_status = ACCESS_ERROR or NOT_AVAILABLE, and let the S1-S7 owning skill decide whether to proceed or request missing details.

If optional context is missing, add it to missing_context and return normally.

If no Confluence-derived campaign details are found after linked Confluence and shared campaign library checks, set shared_confluence_campaign_library_status = NOT_FOUND and include campaign_documentation_evidence in missing_context. Do not fabricate campaign details.
