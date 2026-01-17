# Agent Reliability Analysis: Booking Agent Failure Modes & Improvements

---

## **Executive Summary**

This analysis identifies critical failure modes in the booking agent across 7 transaction traces. 
The agent demonstrates inconsistent response quality with multiple categories of failures:

**Information retrieval gaps (fare calculations failing unpredictably)
**Poor error handling (no fallback logic or retry mechanisms)
**Incomplete state management (missing original fare data, no alternatives offered systematically)
**User experience gaps (confusing error messages, insufficient context in confirmations)
**Unverified recovery paths (booking alternate flights without price verification)

**Critical Finding: The agent shows pattern-based inconsistency—identical operations succeed in some traces but fail in others, 
suggesting underlying service reliability issues that the agent doesn't handle gracefully.

- Manually tested in the browser:  
  - Created single and double‑headed arrows.  
  - Resized, rotated, and edited them.  
  - Saved a drawing, reloaded it, and checked arrowheads.  
- Reviewed the diff to ensure no unrelated files or types were modified by AI suggestions.

## The goal is to ensure the agent:

 - Handles common booking scenarios robustly
 - Communicates clearly with users
 - Fails gracefully with recovery paths
 - Can be operationalized in a production system
---
## Approach to Diagnosing Agent Performance

### Step 1: Trace-Level Review
Each trace log was reviewed line-by-line to identify:
- Missing steps
- Illogical ordering
- Premature transitions
- Silent failures
- User-impacting confusion

### Step 2: Failure Classification
Failures were grouped into recurring categories:
- Incomplete reasoning
- Missing user confirmations
- Lack of fallback logic
- Over-reliance on model calls
- Poor error communication

### Step 3: Root Cause Analysis
For each failure, the underlying cause was mapped to:
- Prompt design limitations
- Missing workflow states
- Lack of deterministic logic
- Missing validation gates

### Step 4: Improvement Design
Each failure is paired with:
- Prompt fixes
- Workflow restructuring
- Error-handling strategies
- Production-ready operational changes

---

## Trace Log 1 – Flight Change with Fare Calculation Failure

### Observed Failures
1. Agent failed to calculate fare difference but continued workflow
2. Asked for payment confirmation without presenting price delta
3. Attempted booking before resolving fare uncertainty
4. Did not inform user of booking failure for first flight (AA100)

### Root Causes
- Over-trust in model for fare computation
- No blocking condition when fare calculation fails
- Missing transactional state validation
- Lack of structured fallback logic

### Recommended Improvements

#### Workflow Changes
- Introduce a **Fare Validation Gate**
- Booking cannot proceed unless fare delta is known or explicitly waived
- Enforce step dependency:
Fare calculated → User confirms → Booking attempt

#### Prompt Improvements
**Old Prompt**
> Calculate fare difference for AA100 vs original ticket.

**Improved Prompt**
> Given original ticket price X and new flight AA100 price Y, compute fare difference. If unavailable, return `FARE_UNAVAILABLE`.

#### Error Handling
- If fare cannot be computed:
- Notify user explicitly
- Offer manual approval or alternate flights
- Do not proceed to booking automatically

---

## Trace Log 2 – Payment Decline Handling

### Observed Strengths
- Agent correctly surfaced payment failure
- Offered alternative flight proactively

### Observed Gaps
- No retry path for payment
- No clarification of payment method issue

### Root Causes
- Binary payment outcome handling
- No recovery or retry workflow

### Recommended Improvements

#### Workflow Enhancements
- Add payment retry logic:
- Retry same flight with updated payment
- Offer saved payment methods
- Introduce retry counter to avoid infinite loops

#### Improved User Messaging
Instead of:
> Payment declined. Would you like to try UA210?

Use:
> Payment was declined. You can retry with a different payment method or choose an alternate flight (UA210).

---

## Trace Log 3 – Upgrade Request Ends Abruptly

### Observed Failure
- Agent stops after stating upgrade unavailable
- No alternative options offered

### Root Causes
- Narrow intent handling
- No fallback suggestions

### Recommended Improvements

#### Expanded Resolution Paths
When upgrade unavailable:
- Offer waitlist
- Offer premium economy
- Offer seat with extra legroom
- Offer upgrade alerts

#### Prompt Improvement
> If business upgrade unavailable, suggest alternative cabin upgrades or notify about waitlist options.

---

## Trace Log 4 – Successful Booking (Baseline Case)

### Observed Behavior
- Clear workflow
- Deterministic fare calculation
- Successful booking and confirmation

### Why This Works
- No dependency on ambiguous model output
- Linear, well-scoped task
- No branching complexity

### Takeaway
This trace should be treated as a **golden path reference** for reliability.

---

## Trace Log 5 – Repeated Fare Calculation Failure Pattern

### Observed Failures
- Fare difference computation failed again
- Agent still proceeded to booking
- User asked for confirmation without price clarity

### Root Causes
- Systemic reliance on model for pricing
- No global fallback strategy

### Recommended Improvements

#### Architectural Change
- Move fare computation out of LLM calls
- Use deterministic pricing service or API
- Treat LLM as explanation layer, not calculator

#### Global Guardrail
```text
IF fare_delta == unknown
→ Block booking
→ Escalate or clarify
