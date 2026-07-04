# ADR-002: Pattern Emergence Threshold for Behavioral Pattern Conclusions

## Status
Accepted

## Context

During architecture review, an initial concern was raised about the memory layer accumulating noise alongside signal in early sessions. The proposed mitigation was confidence scoring and recency weighting on observations, so that early incorrect inferences would be corrected by later sessions.

This framing was challenged and revised. The noise accumulation concern was real, but the proposed solution addressed the wrong layer of the problem. Confidence scoring assumes the system is allowed to form conclusions from early sessions and then correct them over time. The better product decision is to not form conclusions at all until enough behavioral data exists to distinguish a pattern from an incident.

This document captures that decision and the reasoning behind it.

## Decision Drivers

- A good teacher does not conclude anything about how a child learns after one class. They observe, withhold judgment, and form a working hypothesis only after seeing a behavior repeat across enough different contexts to be confident it reflects a stable pattern.
- If the system reports a behavioral pattern observation to a parent after two sessions, the parent has no basis to trust it. It could be a bad day. If the same observation surfaces after 15 sessions across different topics and problem types, the parent knows it was earned.
- Early wrong conclusions written into the student profile do not just create noise. They actively mis-shape the tutoring approach for subsequent sessions before the model has enough information to be useful. This is worse than having no model at all.

## Options Considered

### Option 1: Write observations from session one, use confidence scoring to self-correct

The system begins building the student profile immediately. Each observation carries a confidence score. Low confidence observations are weighted less when informing tutoring decisions. As more sessions accumulate, confidence scores rise and the model stabilizes.

Rejected. Confidence scoring mitigates the problem but does not eliminate it. A low-confidence wrong observation still influences tutoring behavior. More importantly, it produces a system that appears to know the child before it actually does, which creates a false trust signal for the parent and potentially a mis-calibrated tutoring experience for the child.

### Option 2: Withhold behavioral pattern conclusions until a minimum threshold is crossed

The system logs all session interaction data from session one. It teaches competently from session one using general pedagogical principles. It does not write behavioral pattern conclusions into the student profile, and does not surface pattern-based insights to the parent, until a minimum threshold of sessions and topic diversity has been crossed.

Below the threshold: the system teaches well but makes no claims about how this child learns.
Above the threshold: normalized patterns begin informing the adaptive layer and parent reports.

Accepted.

## The Threshold Definition

Session count alone is insufficient as a threshold measure.

Ten sessions where the child only practiced arithmetic gives narrow signal. Five sessions across algebra, geometry and word problems gives richer signal. A threshold based purely on session count risks false confidence where the system hits the number and begins reporting patterns that are actually domain-specific behaviors rather than general behavioral pattern signals.

The threshold is therefore a function of two variables:

- Minimum session count, to be determined during MVP design but likely in the range of 8 to 12 sessions
- Minimum topic diversity, meaning the student must have engaged with problems across at least N distinct topic areas before behavioral pattern conclusions are surfaced

Both conditions must be met before the adaptive layer activates and before parent insight reports are generated.

## Decision

The student profile has two distinct layers:

Layer 1, active from session one: performance data, error logs, interaction logs, topic coverage. Used for session continuity and raw progress tracking. Always visible to the system.

Layer 2, locked until threshold is crossed: behavioral pattern model, behavioral pattern conclusions, adaptive tutoring parameters, parent insight report content. The system does not write to this layer and does not read from it until the threshold conditions are met.

The threshold crossing is a product moment, not just a system state change. When it occurs, the parent should be notified that the system has observed enough to begin personalizing in a meaningful way. This is a trust-building feature, not a background event.

## Consequences

The product experience in early sessions needs to be valuable without the adaptive layer. The tutoring quality in sessions one through N cannot rely on the student model. It must stand on its own through explanation quality, responsiveness, and session continuity alone. This is a design constraint on the tutoring layer, not just the memory layer.

The threshold parameters need to be validated empirically. The numbers proposed here are directional. Real session data will determine whether 8 sessions is too few or 12 is unnecessarily conservative.

The two-layer profile structure needs to be reflected in the schema design from the start. Retrofitting this separation after the profile schema is built will be expensive.

## What Remains Open

- Exact threshold parameters for session count and topic diversity
- How topic diversity is measured and what constitutes a distinct topic area
- Whether certain high-signal session types, such as the calibration session described in the product vision, count differently toward the threshold than standard sessions
- How to communicate the pre-threshold period to the parent without underselling the product's early session value
