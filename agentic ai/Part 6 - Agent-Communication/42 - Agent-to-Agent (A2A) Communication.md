

## Learning Objectives

By the end of this chapter, you will understand:

- What Agent-to-Agent (A2A) Communication is
- Why Agents need to communicate
- How Agents exchange information
- Different communication models
- What a message actually looks like (performatives, not just content)
- Synchronous vs asynchronous communication between Agents
- Delivery guarantees and why they matter
- How Agents authenticate each other
- Challenges in Agent communication
- Best practices for Agent-to-Agent communication

---

# Introduction

Imagine you're building an AI Travel Planner.

Instead of one Agent doing everything,

you have four specialized Agents.

- Flight Agent
- Hotel Agent
- Weather Agent
- Budget Agent

The Flight Agent finds available flights.

How does the Hotel Agent know the travel dates?

How does the Budget Agent know the total flight cost?

The Agents must exchange information.

This process is called **Agent-to-Agent (A2A) Communication**.

Without communication,

multiple Agents cannot work together effectively.

---

# What is Agent-to-Agent (A2A) Communication?

Agent-to-Agent (A2A) Communication is the process by which AI Agents exchange information, tasks, results, and decisions while collaborating to achieve a common goal.

Instead of working independently,

Agents continuously communicate during execution.

Communication is the foundation of every Multi-Agent system.

---

# Why Do Agents Need Communication?

Every Agent has only a partial view of the problem.

To solve a larger task,

Agents need to share information.

Communication allows Agents to:

- Delegate work
- Share knowledge
- Report progress
- Request help
- Coordinate execution
- Build a common understanding of the task

Without communication,

specialized Agents become isolated.

---

# Visual Diagram

```text
                 User

                   │

                   ▼

            Research Agent

                   │
          Research Findings

                   ▼

             Writing Agent

                   │
             Draft Report

                   ▼

             Review Agent

                   │
            Review Feedback

                   ▼

             Final Response
```

Each Agent contributes information to the next stage.

---

# What Do Agents Communicate?

Agents exchange many kinds of information.

Examples include:

- Tasks
- Results
- Tool outputs
- Memory references
- Status updates
- Progress reports
- Errors
- Questions
- Decisions
- Plans

Communication is much more than sending a final answer.

It happens throughout the Agent's lifecycle.

---

# Communication Lifecycle

A typical communication process looks like this.

```text
Create Message

↓

Send Message

↓

Receive Message

↓

Interpret Message

↓

Take Action

↓

(Optional)

Reply
```

Every communication follows this basic lifecycle.

---

# Message Structure: Performatives

A message isn't just its content — it also needs to say **what kind of message it is**. This idea comes from a classic multi-agent standard called FIPA ACL, which defines a set of **performatives** (message intents).

```text
inform    — "Here is a fact." (e.g. "Flight price is $450")
request   — "Please do this." (e.g. "Find hotels for these dates")
propose   — "I suggest this option." (e.g. "How about this itinerary?")
accept    — "I agree to that proposal."
reject    — "I disagree with that proposal."
query     — "What is the value of X?"
failure   — "I couldn't complete that task."
```

Without a performative, a receiving Agent has to guess whether a message is a fact to note, a task to perform, or a question to answer. Tagging the intent explicitly removes that ambiguity.

```json
{
  "performative": "request",
  "sender": "PlannerAgent",
  "receiver": "HotelAgent",
  "content": { "task": "find_hotels", "dates": ["2026-09-01", "2026-09-05"] }
}
```

---

# Communication Models

There are several ways Agents communicate.

---

## 1. One-to-One Communication

One Agent communicates directly with another.

```text
Agent A

↓

Agent B
```

This is the simplest communication model.

---

## 2. One-to-Many Communication

One Agent sends information to several Agents.

```text
             Agent A

         ↙      ↓      ↘

    Agent B  Agent C  Agent D
```

Useful when multiple Agents need the same information.

---

## 3. Many-to-One Communication

Several Agents report to one Agent.

```text
Agent A

↘

Agent B

↘

Agent C

↓

Supervisor
```

This is common in Supervisor Architectures.

---

## 4. Many-to-Many Communication

Every Agent can communicate with every other Agent.

```text
Agent A ↔ Agent B

↑             ↓

Agent C ↔ Agent D
```

This is common in decentralized systems,

but it introduces greater coordination complexity.

---

# Direct vs Indirect Communication

Agents can communicate in two ways.

### Direct Communication

The sender communicates directly with another Agent.

```text
Agent A

↓

Agent B
```

Simple,

but creates tighter coupling.

---

### Indirect Communication

Agents communicate through an intermediate system.

Examples include:

- Shared Memory
- Blackboard
- Event Bus
- Message Queue

```text
Agent A

↓

Shared Workspace

↓

Agent B
```

This improves scalability and flexibility.

---

# Synchronous vs Asynchronous Communication

Not every message needs an immediate reply.

```text
Synchronous
Agent A sends a request and BLOCKS until Agent B responds.
Good for: quick lookups where the answer is needed immediately.

Asynchronous
Agent A sends a request and CONTINUES other work.
Agent B replies later, and Agent A processes the reply when it arrives.
Good for: long-running tasks, or when Agent A has other useful work to do.
```

```text
Agent A: "Find flights"  (async)

↓

Agent A continues checking hotel availability

↓

Agent B eventually replies with flight options

↓

Agent A incorporates the reply once it arrives
```

Choosing synchronous communication everywhere is simpler to reason about but can leave Agents idle waiting on each other unnecessarily — async communication trades that simplicity for better throughput.

---

# Delivery Guarantees

Real networks are unreliable, so a message-passing system has to decide what guarantee it offers.

```text
At-Most-Once
Message is sent once. If it's lost, it's gone. (fastest, least safe)

At-Least-Once
Message is retried until acknowledged.
Risk: the receiver might get it more than once.

Exactly-Once
Message is guaranteed to arrive exactly one time.
(hardest to implement, usually needs deduplication on top of at-least-once)
```

For messages that trigger side effects (like "charge the customer"), **at-least-once with deduplication** (an idempotency key, similar to tool execution) is a common practical compromise — you get reliability without truly needing exactly-once delivery.

---

# Communication During the Agent Loop

Communication is not a one-time activity.

It happens throughout the Agent Loop.

```text
Think

↓

Communicate

↓

Act

↓

Observe

↓

Communicate Again

↓

Continue
```

As new information becomes available,

Agents continue exchanging messages.

---

# Authentication Between Agents

When Agents belong to different systems, teams, or even different companies, a receiving Agent needs to know a message genuinely came from who it claims to be from.

```text
Agent A Sends Message

↓

Message Signed with Agent A's Credentials

↓

Agent B Verifies the Signature

├── Valid   → Trust and Process Message
└── Invalid → Reject Message
```

This matters more as Multi-Agent systems span organizational boundaries — without authentication, a malicious actor could impersonate a trusted Agent and inject false information or unauthorized commands into the system.

---

# Python Example

A simplified example:

```python
message = {
    "sender": "ResearchAgent",
    "receiver": "WriterAgent",
    "task": "Write report",
    "content": "Research completed."
}

print(message)
```

A version including a performative and a message ID for deduplication:

```python
import uuid

message = {
    "message_id": str(uuid.uuid4()),
    "performative": "inform",
    "sender": "ResearchAgent",
    "receiver": "WriterAgent",
    "content": "Research completed.",
}

print(message)
```

Production systems serialize these messages and transmit them over communication channels,

but the concept remains the same.

---

# The A2A Protocol (Industry Standard)

"A2A" isn't just a descriptive term — it's also the name of an actual open protocol (Agent2Agent), originally introduced by Google, aimed at letting Agents built on different frameworks discover each other's capabilities and communicate in a standardized way.

```text
Agent (Framework X)

↓

A2A Protocol (standard message format + discovery)

↓

Agent (Framework Y)
```

The goal is similar to what HTTP does for web servers and browsers built by different vendors — a common protocol so Agents don't need custom, one-off integrations for every other Agent they talk to. It's a complementary idea to MCP: MCP standardizes how an Agent talks to _tools_, while A2A-style protocols standardize how Agents talk to _each other_.

---

# Real-World Example

Imagine an AI Software Development Team.

Workflow

```text
Planner Agent

↓

Backend Agent

↓

Frontend Agent

↓

Testing Agent

↓

Deployment Agent
```

Each Agent communicates:

- Current status
- Completed work
- Errors
- Required changes

Without communication,

the project cannot progress correctly.

If the Testing Agent belongs to a separate internal team's system, its messages to the Deployment Agent would also need to be authenticated, so the Deployment Agent can trust that a "tests passed" message truly came from the real Testing Agent.

---

# Industry Insight

Modern Multi-Agent frameworks rely heavily on structured communication.

Examples include:

- LangGraph state updates
- CrewAI Agent messaging
- AutoGen conversational exchanges
- Google ADK communication workflows
- OpenAI Agents SDK orchestration
- The A2A Protocol for cross-framework Agent communication

Rather than exchanging free-form text,

production systems typically exchange structured messages containing metadata, task information, and execution state.

---

# Best Practices

Use structured communication instead of free-form messages.

Tag messages with a clear intent (performative) rather than relying on the receiver to infer it.

Share only the information another Agent needs.

Keep messages concise and meaningful.

Choose synchronous or asynchronous communication deliberately, based on whether the sender has other useful work to do while waiting.

Define clear communication paths between Agents.

Authenticate messages when Agents cross system or organizational boundaries.

Log important communication for debugging and monitoring.

---

# Common Beginner Mistakes

### Mistake 1

Sending too much information.

Only relevant context should be shared.

---

### Mistake 2

Allowing unrestricted communication between every Agent.

This creates unnecessary complexity.

---

### Mistake 3

Using inconsistent message formats.

Production systems standardize communication.

---

### Mistake 4

Assuming communication is always reliable.

Messages may be delayed, lost, duplicated, or arrive out of order.

Production systems must handle these situations.

---

### Mistake 5

Blocking synchronously on every message.

If the sender has other useful work available, synchronous-only communication wastes that time unnecessarily.

---

### Mistake 6

Trusting every incoming message without verifying its sender.

Especially across system boundaries, an unauthenticated message could be spoofed or tampered with.

---

# Interview Tip

A common interview question is:

> **What is Agent-to-Agent (A2A) Communication?**

A good answer is:

Agent-to-Agent (A2A) Communication is the process by which AI Agents exchange information, tasks, results, and decisions while collaborating to solve a problem. It enables coordination, specialization, and efficient execution in Multi-Agent systems.

A strong follow-up point: mention that messages typically carry an explicit intent (a performative, borrowed from the classic FIPA ACL standard), that communication can be synchronous or asynchronous with different delivery guarantees, and that "A2A" also refers to an actual emerging open protocol for cross-framework Agent communication.

---

# Where is this Used?

- LangGraph
- CrewAI
- AutoGen
- Google ADK
- OpenAI Agents SDK
- Enterprise Multi-Agent Platforms
- Cross-Framework Agent Systems via the A2A Protocol

---

# Key Takeaways

- Agent-to-Agent Communication enables collaboration between AI Agents.
- Agents exchange tasks, results, progress, errors, and decisions.
- Messages should carry an explicit intent (performative) — inform, request, propose, and so on.
- Communication can be one-to-one, one-to-many, many-to-one, or many-to-many.
- Communication may be direct or indirect, and synchronous or asynchronous.
- Delivery guarantees (at-most-once, at-least-once, exactly-once) affect reliability trade-offs.
- Authentication matters when Agents span different systems or organizations.
- A2A is also the name of a real open protocol standardizing cross-framework Agent communication.
- Well-designed communication is essential for scalable Multi-Agent systems.

---


