
## Learning Objectives

By the end of this chapter, you will understand:

- What Agent Permissions are
- Why AI Agents need permissions
- Types of Agent Permissions
- Permission models used in AI systems
- Risks of excessive permissions
- Best practices for managing Agent Permissions
- Why a permission check enforced by your system is fundamentally different from a prompted restriction — and why only one of them is real security
- Why an agent generally shouldn't have more access than the user it's acting on behalf of
- How dynamic, scoped, time-limited permissions differ from static permission tables

---

# Introduction

Imagine you give an AI Agent access to your email.

You ask it:

```text
Summarize today's emails.
```

To complete this task,

the Agent only needs permission to:

- Read emails

But what if the Agent also has permission to:

- Delete emails
- Send emails
- Access your Google Drive
- Modify your calendar

Those permissions are unnecessary.

If the Agent makes a mistake,

or is compromised,

it could perform actions you never intended.

This is why AI systems use **Agent Permissions**.

---

# What are Agent Permissions?

Agent Permissions define **what actions an AI Agent is allowed to perform**.

Permissions act as security boundaries.

Instead of giving an Agent unlimited access,

the system grants only the capabilities required for its task.

---

# The Most Important Distinction: Prompted Restriction vs Enforced Permission

Before going further, this is worth understanding clearly — it's arguably the single most important security concept in this entire chapter, and a favorite deeper-interview question.

There's a critical difference between two things that can look similar on the surface:

```text
Prompted restriction (NOT real security)
→ "You are not allowed to delete emails." written in the system prompt
→ Relies entirely on the model choosing to comply
→ Can be bypassed by a confused model, a bug, or a prompt injection
  attack that convinces the model the restriction doesn't apply here

Enforced permission (ACTUAL security)
→ The delete_email function simply doesn't exist in this agent's
  available tools, OR
→ The function exists, but the application code checks permissions
  BEFORE executing it, and rejects the call regardless of what the
  model requested
```

The model's own "good behavior" is never a security boundary — it's a preference, not a guarantee. This connects directly back to the "never blindly execute what the model requests" point from the Function Calling chapter: a permission system's entire purpose is to be the backstop that works _even when_ the model is wrong, confused, or manipulated. If the only thing stopping a destructive action is a sentence in the prompt, that isn't really a permission system — it's a suggestion.

---

# Why Do Agents Need Permissions?

AI Agents interact with many external systems.

Examples include:

- Email
- Databases
- Cloud Storage
- Calendars
- Payment Systems
- Internal APIs
- MCP Servers

Without permissions,

an Agent could accidentally or maliciously perform dangerous actions.

Permissions reduce this risk.

---

# Visual Diagram

```text
            AI Agent

                │

        Permission Check

                │

      ┌─────────┴─────────┐

      ▼                   ▼

Allowed Action      Blocked Action
```

Every sensitive operation should be checked before execution.

---

# Types of Permissions

Different Agents require different permissions.

## Read Permission

Allows the Agent to retrieve information.

Example

```text
Read Emails

Read Files

Read Calendar
```

---

## Write Permission

Allows the Agent to create or modify information.

Example

```text
Create Document

Update Database

Write File
```

---

## Delete Permission

Allows the Agent to remove resources.

Example

```text
Delete Email

Delete File

Delete Database Record
```

This permission should be granted carefully.

---

## Execute Permission

Allows the Agent to execute tools or commands.

Example

```text
Run Python Script

Call External API

Execute Shell Command
```

---

## Administrative Permission

Allows high-privilege operations.

Example

```text
Manage Users

Change Permissions

Deploy Applications
```

These permissions should be restricted to trusted Agents.

---

# Permission Workflow

Every action follows a similar process.

```text
Agent Requests Action

↓

Permission Check

↓

Allowed?

↓

Yes

↓

Execute

--------------------

No

↓

Reject Action
```

The Agent never bypasses the permission check.

---

# Example

Imagine an Email Assistant.

Allowed

```text
Read Emails

Summarize Emails
```

Not Allowed

```text
Delete Emails

Send Emails
```

The Agent can complete its task without unnecessary permissions.

---

# An Agent Shouldn't Have More Access Than the User It Acts For

This is a specific, commonly overlooked risk worth calling out directly: agents typically act **on behalf of** a specific user, and their effective permissions should never exceed what that user is personally allowed to do.

```text
Bad design
→ Agent runs with a broad "service account" that has admin-level access
  to everything, regardless of which user it's currently helping

Better design
→ Agent's permissions are scoped to (and never exceed) the permissions
  of the specific user it's currently acting for
```

Why this matters: if an agent runs with elevated, shared credentials instead of the requesting user's actual permissions, then any flaw in the agent (a bug, a successful prompt injection, a hallucinated action) can let a low-privilege user's request cause a high-privilege action — essentially a privilege escalation bug, just introduced through an AI system instead of traditional application code. This is sometimes called the **"confused deputy" problem**: the agent, like a deputy acting with more authority than the person it serves, can be tricked into misusing its own elevated access.

---

# Permission Models

Production AI systems commonly use these models.

## Role-Based Access Control (RBAC)

Permissions are assigned based on the Agent's role.

Example

```text
Support Agent

↓

Read Customer Tickets

--------------------

Finance Agent

↓

Access Billing Records
```

---

## Attribute-Based Access Control (ABAC)

Permissions depend on attributes such as:

- User
- Resource
- Time
- Location
- Environment

Example

```text
Allow Access

↓

Only During Business Hours
```

---

## Policy-Based Access

Permissions are controlled using policies.

Example

```text
May Read Documents

↓

Cannot Delete Documents
```

Policies provide fine-grained control over Agent behavior.

---

# Dynamic, Scoped, and Time-Limited Permissions

The examples so far show fairly static permission tables (this agent can read, this agent can't delete). Real production systems often go further, using permissions that are dynamic rather than fixed.

```text
OAuth Scopes
→ When a user connects a third-party service (e.g. Gmail, Google Drive),
  they typically grant a specific, limited SCOPE of access
  (e.g. "read-only access to Calendar events"), not blanket account access

Time-Limited Tokens
→ Access grants that automatically expire after a set period, requiring
  re-authorization rather than remaining valid forever

Session-Scoped Permissions
→ An agent might be granted elevated permissions only for the duration
  of a specific task or session, reverting to minimal access afterward
```

This connects to earlier chapters on tool design: rather than a single static "can this agent delete files: yes/no" flag, real systems often ask "does this specific token, for this specific user, in this specific session, still have a valid, unexpired grant for this specific scope of action?" — a much finer-grained and safer default.

---

# Human-in-the-Loop for Sensitive Actions

This connects directly to the "require confirmation for sensitive actions" point from the Function Calling chapter — worth restating here as a permissions-specific pattern.

```text
Low-risk action (read a file)     → execute automatically
Medium-risk action (send an email) → maybe execute automatically,
                                      with logging
High-risk action (delete records,
transfer money, deploy code)       → require explicit human
                                      confirmation before executing,
                                      even if the agent technically
                                      has permission
```

Having permission to do something and being allowed to do it _without a human checking first_ are two different decisions. The riskier and harder to reverse an action is, the more it benefits from a human confirmation step as an additional layer, on top of (not instead of) the underlying permission system.

---

# Python Example

A simplified example:

```python
permissions = {
    "read_email": True,
    "delete_email": False
}

if permissions["read_email"]:
    print("Reading emails...")

if not permissions["delete_email"]:
    print("Delete operation denied.")
```

Production systems integrate permission checks with authentication and authorization services.

Note: this check happens in application code, after the model has already responded — this is exactly the enforced-permission pattern described earlier, as opposed to relying on the model choosing not to request the action in the first place.

---

# Real-World Example

Imagine an Enterprise HR Assistant.

Allowed

```text
Read Employee Records

Generate Reports
```

Blocked

```text
Modify Payroll

Delete Employee Records

Change User Permissions
```

The Agent receives only the permissions necessary for its responsibilities.

---

# Industry Insight

Modern Agent platforms implement permission systems before every tool call.

Examples include:

- OpenAI Agents SDK tool authorization
- Google ADK access policies
- MCP capability permissions
- Enterprise IAM (Identity and Access Management)
- OAuth-based authorization

Permissions are a fundamental layer of production AI security.

---

# Best Practices

Grant only the permissions required for the current task.

Review permissions regularly.

Separate read and write permissions.

Require additional approval for sensitive operations.

Log permission usage for auditing.

Never rely on a prompt instruction alone as your only line of defense — enforce permissions in application code.

Scope an agent's permissions to, and never beyond, the user it's acting on behalf of.

---

# Common Beginner Mistakes

### Mistake 1

Giving every Agent administrator access.

Most Agents require only limited permissions.

---

### Mistake 2

Granting permanent permissions.

Permissions should be reviewed and revoked when no longer needed.

---

### Mistake 3

Skipping permission checks.

Every sensitive action should be verified before execution.

---

### Mistake 4

Combining unrelated permissions.

Different Agent roles should have different permission sets.

---

### Mistake 5

Believing that telling the model "don't do X" in the prompt is a real permission boundary.

It isn't — it's a preference the model might not reliably follow, especially under prompt injection or confusion. Real boundaries are enforced in code, not requested in a prompt.

---

### Mistake 6

Running an agent with broader access than the specific user it's currently serving.

This creates a confused-deputy risk, where a low-privilege user's request can trigger a high-privilege action through the agent.

---

# Interview Tip

A common interview question is:

> **Why are Agent Permissions important?**

A good answer is:

Agent Permissions define the actions an AI Agent is allowed to perform. They restrict access to sensitive resources, reduce security risks, prevent accidental misuse, and ensure that Agents operate only within their authorized capabilities.

---

# Interview Tip

A stronger follow-up worth preparing for:

> **If you tell an agent in its system prompt "never delete files," is that a sufficient security control? Why or why not?**

Answer:

No. A prompt instruction is a preference the model may follow, but it isn't an enforced boundary — a confused model, a bug, or a prompt injection attack could still cause the model to request the disallowed action. Real permission enforcement has to happen in application code: either the tool isn't available to the agent at all, or a permission check runs before execution and rejects the request regardless of what the model asked for. The model's compliance should never be the only safeguard for a genuinely sensitive action.

---

# Where is this Used?

- OpenAI Agents SDK
- Google ADK
- MCP
- Enterprise IAM Systems
- OAuth
- Production AI Platforms

---

# Key Takeaways

- Agent Permissions define what an AI Agent can and cannot do.
- There's a critical difference between a prompted restriction (a preference the model might not follow) and an enforced permission (a real boundary checked in application code) — only the latter is real security.
- Permissions should be checked before every sensitive action, in code, not just requested in the prompt.
- An agent's permissions should never exceed those of the user it's acting on behalf of, to avoid confused-deputy risks.
- Common permission types include read, write, delete, execute, and administrative access.
- Production systems commonly use RBAC, ABAC, and policy-based authorization, often made dynamic with OAuth scopes and time-limited tokens rather than static permission tables.
- Sensitive, high-risk, or hard-to-reverse actions benefit from human-in-the-loop confirmation, even when the agent technically has permission.
- Well-designed permission systems reduce security risks and improve reliability.

---

