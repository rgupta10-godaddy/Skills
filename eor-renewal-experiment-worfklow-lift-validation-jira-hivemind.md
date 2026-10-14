Lift Validation Using Jira and Hivemind
Scope
This skill applies only when workflow_intent = size_opportunity. Do not use this skill for setup-experiment or gather-results workflows.

Purpose
Derive lift, validate it using completed Jira test proof points and completed Hivemind experiment proof points, score evidence quality, and confirm lift before sizing.

Actual measured outcomes carry more weight than similar lift-level hypotheses. Highest-priority evidence is Jira customfield_17006 and Hivemind final measured results.

Only use completed Jira tests and completed Hivemind experiments as proof points. Hivemind search must not be constrained to same business unit, campaign, project, or Jira namespace.

Hypothesis-only expected lifts are context only and must not be used in observed-lift averages.

Historical validation limits
Return a maximum of 5 comparable experiments. Stop searching once 5 confidence-worthy analogs are found.

Do not continue searching for additional matches after the confidence score can be calculated with 5 experiments.

Rationale:
Discovery across Jira/Hivemind for comparable experiments creates unnecessary retrieval churn. The confidence calculation can be performed with 5 high-quality analogs. Additional searches beyond 5 experiments waste tokens without improving confidence.

Lift change tracking
If the user changes, revises, overrides, or replaces the proposed lift during this validation gate:

Store the original proposed lift as lift_value_original.
Store the final confirmed lift as lift_value_confirmed.
Store the confirmed lift type as lift_type_confirmed.
Set lift_changed_by_user = true.
Tell the user that the final Jira writeback dry run will include an update to the Hypothesis field customfield_24203 to reflect the final confirmed lift assumption.
Confidence threshold auto-proceed
When confidence >= 90%, assume inferred lift is correct and proceed without user confirmation.

When confidence < 90%, request user confirmation before proceeding.

Confidence scoring:

High confidence (>= 90%): 3+ completed experiments with measured results, or Jira test proof with measured lift value
Medium confidence (50-89%): 1-2 completed experiments, or hypothesis-only Jira evidence
Low confidence (< 50%): No direct proof points, inferred from campaign context only
Auto-proceed behavior (confidence >= 90%):

Set lift_confirmed = true automatically
Set sizing_s8_complete = true
Set current_gate = "S9"
Proceed to S9-S10 without user interaction
Display confirmation notice: "Lift validated with high confidence (X%), proceeding to sizing."
Confirmation prompt (confidence < 90% only):

Based on the Jira ticket, completed Jira test proof points, and completed Hivemind experiment proof points, I found the proposed lift estimate below. Confidence is below 90%, please confirm whether I should use this lift or provide a revised estimate.

- Proposed lift: <value>
- Lift type: <absolute or relative>
- Confidence: <percentage>%

Reply confirmed to proceed with this lift, or provide a revised lift and state whether it is absolute or relative.
Rationale:
The workflow currently includes a confirmation loop for every lift validation, even when evidence is strong. This removes a full interaction turn and associated tokens when confidence is high, while preserving the confirmation gate when evidence is weak.

Completion requirements
When lift is confirmed:

Set lift_confirmed = true
Set sizing_s8_complete = true
Set current_gate = "S9"
Return control to master orchestrator
Master orchestrator will automatically proceed to S9-S10 (Helix sizing) without asking user what to do next.
