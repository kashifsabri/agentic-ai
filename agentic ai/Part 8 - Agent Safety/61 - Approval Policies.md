

These notes fill in concepts that are commonly expected alongside Approval Policies but were missing from the base chapter. Same style as the original — add this as extra sections, don't replace anything.

---

# Deny-by-Default

The base chapter's example policy assumes every action has a rule. Real systems need to handle the action that isn't listed at all.

```text
Action Requested

↓

Does a Policy Rule Exist?

↓

Yes -> Follow the Rule

↓

No -> DENY by default
```

A missing rule should never be treated as automatic approval.

```text
Bad

↓

Unknown action -> Allow ("we'll add a rule later")

Good

↓

Unknown action -> Deny + Alert policy owner
```

This protects against Agents calling new or unexpected tools that nobody has classified yet.

---

# Static Risk Classification vs Dynamic Risk Scoring

The base chapter classifies actions into fixed Low / Medium / High buckets. Production systems often go further.

```text
Static Classification

↓

"delete_file" is ALWAYS High Risk,
 regardless of context
```

```text
Dynamic Risk Scoring

↓

Risk = f(action, resource, user, volume, time, history)

↓

"delete_file" on 1 test file    -> Low
"delete_file" on 5,000 files    -> High
"delete_file" at 3am, new Agent -> High
```

The same action type can carry different risk depending on scale, timing, and who is asking — a static table can't capture this.

---

# Attribute-Based Policies (ABAC)

An extension of the "user permissions" factor mentioned in the base chapter's decision inputs.

```text
Role-Based Rule

↓

"Managers can approve refunds"
```

```text
Attribute-Based Rule

↓

"Approve refunds under ₹5,000
 IF requester.department == customer.department
 AND time == business_hours
 AND requester.trust_score > 80"
```

ABAC policies combine multiple attributes instead of relying on a single role label. This allows finer-grained rules than plain role-based approval.

---

# Policy-as-Code

The base chapter says "keep approval policies separate from Agent logic." Here's the common way that's implemented.

```text
Policy Engine (separate service)

↓

Rules written in a dedicated policy language
 (not embedded in prompts, not hardcoded in Agent code)

↓

Agent sends: "Can I do X, on Y, as user Z?"

↓

Policy Engine returns: allow / require_approval / deny
```

Benefits

```text
Rules can be tested independently of the Agent

Rules can be version-controlled like code

The same engine can serve many different Agents
```

---

# Policy Conflict Resolution

With many rules, two rules can sometimes disagree.

```text
Rule A: "HR Agents can read employee records" -> Allow

Rule B: "Salary data always requires approval" -> Require Approval

↓

Conflict: reading a salary field
```

Common resolution strategies

```text
Most Restrictive Wins
 -> If any rule says "require approval" or "deny",
    that rule wins over an "allow"

Most Specific Wins
 -> A rule about "salary field" beats
    a general rule about "employee records"
```

Most-restrictive-wins is the safer default for security-sensitive systems.

---

# Separation of Duties

A policy design principle not mentioned in the base chapter.

```text
The person who REQUESTS an action

↓

should not be the same person who APPROVES it
```

Example

```text
Agent prepares a payment on behalf of Employee A

↓

Employee A cannot also be the approver

↓

Approval must come from someone else
```

This prevents a single compromised account (human or Agent) from both requesting and approving a high-risk action.

---

# Break-Glass Procedures

Policies sometimes need a documented emergency override.

```text
Normal Policy

↓

"Delete Production Database" -> Always Blocked
```

```text
Break-Glass Path

↓

Requires: 2 senior approvers + incident ticket + full audit log

↓

Action allowed ONLY through this emergency path,
 never through the Agent's normal flow
```

Break-glass access should be rare, heavily logged, and reviewed after every use — it is an exception process, not a shortcut.

---

# Cumulative / Chained Action Risk

A single action can look safe. A sequence of them might not.

```text
Action 1: Read customer email        -> Low Risk
Action 2: Read customer phone number -> Low Risk
Action 3: Read customer address      -> Low Risk

↓

Combined: Full customer profile assembled -> High Risk
```

Some policy engines track risk **across a session**, not just per action, so that many "low risk" steps don't quietly add up to one high-risk outcome.

---

# Testing Policies Before Deployment

The base chapter says "review approval policies regularly." Testing is a related but distinct practice.

```text
New Policy Written

↓

Run Against Historical Requests (simulation)

↓

Check: Did any high-risk action get auto-approved?
Check: Did routine low-risk actions get blocked unnecessarily?

↓

Fix Issues

↓

Deploy
```

This catches policy bugs — like an overly broad "allow" rule — before they reach production Agents.

---

# Policy Versioning and Rollback

Policies change over time and need the same discipline as code.

```text
Policy v1 -> Deployed

↓

Policy v2 -> Deployed (new rule added)

↓

Issue Detected (v2 blocks too much / allows too much)

↓

Rollback to v1

↓

Investigate before re-deploying v2
```

Without version history, it's hard to know what rule was active when a specific past action was approved.

---

# Handling Policy Engine Failures

What happens if the policy engine itself is unreachable?

```text
Agent Requests Decision

↓

Policy Engine Times Out / Is Down

↓

Fail Closed  -> Treat as Deny (safe)

  vs.

Fail Open    -> Treat as Allow (dangerous)
```

Fail Closed should be the default for any action above Low Risk — a broken policy engine should never silently grant access.

---

# Updated Best Practices (Additions)

Default to deny for any action without an explicit rule.

Use dynamic risk scoring where possible, not just a fixed per-action label.

Apply "most restrictive wins" when two rules conflict.

Enforce separation of duties between requesters and approvers.

Track cumulative risk across a session, not only per individual action.

Simulate new policies against historical data before deploying them.

Version policies and keep the ability to roll back.

Fail closed if the policy engine is unavailable.

---

# Updated Common Beginner Mistakes (Additions)

### Mistake 5

Allowing an action by default when no rule matches it.

---

### Mistake 6

Letting the same actor both request and approve a sensitive action.

---

### Mistake 7

Deploying policy changes without testing them against past requests first.

---

### Mistake 8

Configuring the system to "fail open" (allow everything) when the policy engine is down.

---

### Mistake 9

Scoring risk purely by action type, ignoring volume, timing, or requester history.

---

# Updated Key Takeaways (Additions)

- Policies should deny unmatched actions by default, not allow them.
- Dynamic risk scoring captures context that static risk labels miss.
- Conflicting rules should resolve toward the more restrictive outcome.
- Separation of duties keeps one actor from both requesting and approving.
- A sequence of low-risk actions can add up to a high-risk outcome.
- Policies need testing, versioning, and a safe (fail-closed) failure mode, just like any other production system.

---

