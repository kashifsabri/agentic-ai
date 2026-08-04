

## Learning Objectives

By the end of this chapter, you will understand:

- What Shared Memory is
- Why Multi-Agent systems use Shared Memory
- How Shared Memory works
- Different types of Shared Memory
- How to scope memory so Agents don't interfere with unrelated tasks
- Access control: who can read or write which parts of Shared Memory
- The difference between Shared Memory and an Agent's context window
- How to choose a storage backend for Shared Memory
- Advantages and limitations of Shared Memory
- Best practices for managing Shared Memory

---

# Introduction

Imagine a team working on a project.

Instead of emailing each other every few minutes,

they use a shared document.

Anyone can:

- Read updates
- Add information
- Track progress
- Continue someone else's work

Everyone always sees the latest version.

AI Agents often work the same way.

Instead of constantly exchanging messages,

they share information through a common memory.

This is called **Shared Memory**.

---

# What is Shared Memory?

Shared Memory is a common storage area that multiple AI Agents can read from and write to while working on the same task.

Instead of sending every piece of information through messages,

Agents can store important information in Shared Memory,

where other Agents can access it whenever needed.

---

# Why Do Agents Need Shared Memory?

Imagine three Agents working together.

- Research Agent
- Writing Agent
- Review Agent

Without Shared Memory,

they must continuously exchange messages.

```text
Research

↓

Writer

↓

Reviewer

↓

Writer

↓

Research
```

Communication quickly becomes complex.

With Shared Memory,

every Agent reads and writes to the same workspace.

```text
          Shared Memory

         ▲      ▲      ▲

 Research  Writer  Reviewer
```

This simplifies collaboration.

---

# Visual Diagram

```text
              Shared Memory
        ┌────────────────────┐
        │ Research Results   │
        │ Draft Report       │
        │ Review Comments    │
        │ Task Status        │
        └────────────────────┘

          ▲       ▲       ▲

     Research  Writer  Reviewer
```

Every Agent accesses the same shared information.

---

# What is Stored in Shared Memory?

Shared Memory may contain:

- Intermediate results
- Task status
- Plans
- Tool outputs
- Observations
- Shared knowledge
- Progress updates

It usually stores information needed by multiple Agents.

---

# Shared Memory Workflow

A typical workflow looks like this.

```text
Agent A

↓

Write Information

↓

Shared Memory

↓

Agent B

↓

Read Information

↓

Continue Task
```

Agents collaborate by sharing information through memory.

---

# Types of Shared Memory

## 1. Temporary Shared Memory

Used only while a task is running.

Example

```text
Current Plan

Current Status

Intermediate Results
```

The memory is cleared after the task completes.

---

## 2. Persistent Shared Memory

Information remains available after execution finishes.

Example

```text
Customer Preferences

Previous Conversations

Project History
```

Future Agents can reuse this information.

---

## 3. Structured Shared Memory

Information is stored in predefined fields.

Example

```json
{
    "task": "Write Report",
    "status": "Completed",
    "owner": "WriterAgent"
}
```

Easy to search and validate.

---

## 4. Unstructured Shared Memory

Information is stored as documents,

notes,

or free-form text.

Example

```text
Research notes

Meeting summary

Project documentation
```

More flexible,

but harder to search.

---

# Scoping Shared Memory: Per-Task vs Global

Not all Shared Memory should be visible to everyone all the time — scope matters.

```text
Global Scope
Visible to every Agent, across every task
(e.g. company-wide policies, shared reference data)

Session Scope
Visible only to Agents working on the current user's conversation
(e.g. this user's preferences for this chat)

Task Scope
Visible only to Agents collaborating on one specific task
(e.g. intermediate results for "generate this specific report")
```

```text
Task A's Agents write to task_a:*

Task B's Agents write to task_b:*

↓

Task A's Agents Never Accidentally Read Task B's Data
```

Without clear scoping (often implemented as key namespacing, e.g. `task_id:field_name`), unrelated tasks running concurrently can leak information into each other — one user's data showing up in another user's session is a serious bug, not just an inconvenience.

---

# Access Control: Who Can Read/Write What

Beyond scoping by task, individual fields often need their own read/write rules.

```text
Field: "financial_analysis"
- Write access: Financial Agent only
- Read access: Financial Agent, Risk Agent, Summary Agent

Field: "draft_report"
- Write access: Writer Agent, Review Agent
- Read access: All Agents
```

```text
Agent Attempts to Write

↓

Does This Agent Have Write Permission for This Field?

├── Yes → Write Succeeds

└── No  → Write Rejected
```

This prevents an unrelated Agent from accidentally (or maliciously) overwriting data it doesn't own — the same principle as least-privilege access in tool security.

---

# Shared Memory vs Message Passing

These concepts are related,

but they solve different problems.

|Message Passing|Shared Memory|
|---|---|
|Agents send messages|Agents read and write shared data|
|Best for notifications|Best for collaboration|
|Temporary communication|Shared workspace|
|Event-based|State-based|

Many production systems use both together.

---

# Shared Memory vs an Agent's Context Window

It's worth being precise about how Shared Memory relates to what the LLM actually "sees."

```text
Shared Memory (external storage)
Can hold large amounts of data — megabytes of history, results, documents.

↓

Only the RELEVANT PORTION is pulled into the LLM's Context Window
(a limited number of tokens the model can process at once)

↓

LLM Reasons Only Over What Was Loaded, Not the Entire Shared Memory
```

This distinction matters: Shared Memory doesn't bypass context window limits — it just means the full dataset lives outside the prompt, and each Agent selectively loads (and often summarizes) the parts it needs, rather than the LLM having to read everything at once.

---

# Memory Consistency

If multiple Agents update Shared Memory simultaneously,

problems may occur.

Example

```text
Agent A

Status = Completed

--------------------

Agent B

Status = In Progress
```

Which value is correct?

The system must maintain **memory consistency**.

Common techniques include:

- Locking
- Versioning
- Transactions
- Conflict detection

These ensure Shared Memory remains accurate.

---

# Consistency vs Availability in Distributed Shared Memory

When Shared Memory is spread across multiple servers (for scale or reliability), a fundamental trade-off appears.

```text
Strong Consistency
Every Agent always sees the exact latest write.
→ May have to wait if data is being synchronized across servers.

Eventual Consistency
An Agent might briefly see slightly stale data.
→ Faster and more available, but requires tolerating brief staleness.
```

```text
Agent A Writes to Server 1

↓

Server 1 Syncs to Server 2 (takes a moment)

↓

Agent B Reads from Server 2

├── Strong Consistency  → Agent B waits until sync completes
└── Eventual Consistency → Agent B might read the old value briefly
```

For most Agent coordination, eventual consistency is an acceptable trade-off — but for anything tied to a critical decision (like "has this payment already been approved?"), strong consistency is usually worth the extra latency.

---

# Expiration Policies (TTL)

Not everything in Shared Memory should live forever — most entries are only useful for a limited window.

```text
Entry Written

↓

Time-To-Live (TTL) Assigned
(e.g. "expire in 1 hour" or "expire when task completes")

↓

TTL Reached

↓

Entry Automatically Removed
```

This keeps Shared Memory from growing unbounded with stale intermediate results long after a task has finished — directly addressing the "treating Shared Memory as permanent storage" mistake.

---

# Choosing a Storage Backend

The right backend depends on scale, persistence needs, and access patterns.

|Backend|Best For|Trade-off|
|---|---|---|
|In-memory dict/object|Prototyping, single-process systems|Lost on restart, doesn't scale across machines|
|Redis (in-memory cache/store)|Fast, temporary shared state across processes|Needs its own infrastructure; typically not the system of record|
|Relational DB (e.g. PostgreSQL)|Structured, persistent, queryable data|Slower than in-memory options|
|Vector Database|Semantic search over shared knowledge/documents|Not ideal for simple key-value state|

Many production systems combine more than one: Redis for fast, short-lived coordination data, and a database or vector store for anything that needs to persist or be searched semantically.

---

# Python Example

A simplified example:

```python
shared_memory = {}

shared_memory["research"] = "Market analysis completed."

print(shared_memory["research"])
```

A version reflecting task scoping and a TTL-style expiration hint:

```python
shared_memory = {}

task_id = "task_42"

shared_memory[f"{task_id}:research"] = {
    "value": "Market analysis completed.",
    "written_by": "ResearchAgent",
    "expires_in_seconds": 3600
}

print(shared_memory[f"{task_id}:research"])
```

Multiple Agents can access the same data structure.

Production systems often use databases,

distributed caches,

or state stores instead.

---

# Real-World Example

Imagine an AI Software Development Platform.

Workflow

```text
Planner Agent

↓

Shared Memory

↓

Backend Agent

↓

Shared Memory

↓

Testing Agent

↓

Shared Memory

↓

Documentation Agent
```

Every Agent reads the latest project state before starting work.

Each project's data is scoped under its own `project_id`, so two different development teams running the platform simultaneously never see each other's intermediate results.

---

# Industry Insight

Most production Multi-Agent systems include some form of Shared Memory.

Examples include:

- LangGraph Shared State
- Blackboard Architectures
- Distributed State Stores
- Redis
- PostgreSQL
- Vector Databases for shared knowledge

Shared Memory reduces unnecessary communication and helps Agents maintain a consistent view of the task.

Frameworks increasingly build in scoping (per-session or per-thread state) by default, precisely because unscoped global memory becomes a common source of cross-task data leaks as systems grow.

---

# Best Practices

Store only information that multiple Agents need.

Scope memory by task or session to prevent cross-task data leaks.

Define read/write access per field, not just per Agent.

Remove outdated or temporary information — use TTLs where appropriate.

Use structured formats whenever possible.

Protect Shared Memory from conflicting updates.

Choose a storage backend based on actual scale and persistence needs.

Clearly define which Agents can modify specific data.

---

# Common Beginner Mistakes

### Mistake 1

Treating Shared Memory as permanent storage.

Temporary execution data should be removed when it is no longer needed — TTLs help enforce this automatically.

---

### Mistake 2

Allowing every Agent to modify every field.

Define ownership and access rules.

---

### Mistake 3

Ignoring concurrent updates.

Multiple Agents may modify Shared Memory at the same time.

---

### Mistake 4

Storing duplicate information.

Avoid unnecessary duplication to keep Shared Memory consistent.

---

### Mistake 5

Using one global, unscoped memory space for every task.

Without scoping, concurrent tasks or users can leak data into each other.

---

### Mistake 6

Assuming Shared Memory automatically fits in the LLM's context.

An Agent still needs to selectively load and summarize the relevant portion — Shared Memory doesn't remove context window limits.

---

# Interview Tip

A common interview question is:

> **What is Shared Memory in a Multi-Agent system?**

A good answer is:

Shared Memory is a common storage area that multiple AI Agents use to read and write shared information during execution. It reduces unnecessary communication, improves collaboration, and helps Agents maintain a consistent understanding of the current task.

A strong follow-up point: mention **scoping** (task/session vs global) to prevent data leaks between concurrent tasks, and that Shared Memory is external storage — only the relevant portion is loaded into an Agent's limited context window, not the entire dataset at once.

---

# Where is this Used?

- LangGraph
- CrewAI
- AutoGen
- Blackboard Architectures
- Distributed AI Systems
- Enterprise Multi-Agent Platforms

---

# Key Takeaways

- Shared Memory is a common workspace for multiple AI Agents.
- Agents collaborate by reading from and writing to shared information.
- Shared Memory may be temporary or persistent.
- Scoping (task, session, or global) prevents unrelated tasks from leaking data into each other.
- Access control defines which Agents can read or write specific fields.
- Shared Memory is separate from an Agent's context window — only relevant parts get loaded into a prompt.
- Distributed Shared Memory involves a trade-off between strong and eventual consistency.
- TTLs/expiration policies keep memory from growing unbounded with stale data.
- Memory consistency is essential when multiple Agents update the same data.
- Production systems often combine Shared Memory with Message Passing.

---

