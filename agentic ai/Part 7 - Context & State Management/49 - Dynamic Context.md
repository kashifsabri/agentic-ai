

## Learning Objectives

By the end of this chapter, you will understand:

- What Dynamic Context is
- Why AI Agents need Dynamic Context
- How Dynamic Context changes during execution
- The components of Dynamic Context
- How to manage context growth and window limits
- How Dynamic Context works in multi-agent systems
- Advantages and challenges of Dynamic Context
- Best practices for managing Dynamic Context

---

# Introduction

Imagine you ask an AI Agent:

```text
Plan a 5-day trip to Japan.
```

At the beginning,

the Agent knows only your request.

After a few seconds,

it has gathered:

- Flight information
- Hotel options
- Weather forecasts
- Tourist attractions

The information available to the Agent has changed.

As the Agent continues working,

its understanding of the task keeps evolving.

This continuously changing information is called **Dynamic Context**.

---

# What is Dynamic Context?

Dynamic Context is the collection of information that changes while an AI Agent is solving a task.

As new information becomes available,

the Agent updates its context.

Unlike a fixed prompt,

Dynamic Context continuously evolves during execution.

---

# Why Do Agents Need Dynamic Context?

At the beginning of a task,

the Agent knows very little.

As it works,

it collects:

- User inputs
- Tool results
- Memory retrievals
- Intermediate reasoning
- Agent outputs

Every new piece of information helps the Agent make better decisions.

---

# Visual Diagram

```text
User Request

↓

Initial Context

↓

Tool Results

↓

Updated Context

↓

Memory Retrieved

↓

Updated Context

↓

Agent Decisions

↓

Final Context

↓

Response
```

The context grows and changes throughout execution.

---

# What Makes Context Dynamic?

Several events can update the context.

Examples include:

- User sends a new message.
- A tool returns new information.
- Memory retrieval finds relevant knowledge.
- Another Agent shares results.
- The Planner creates a new plan.
- The Executor completes a task.

Each event changes what the Agent knows.

---

# Components of Dynamic Context

Dynamic Context typically includes:

- Current user request
- Conversation history
- Retrieved memories
- Tool outputs
- Intermediate plans
- Agent observations
- Current task status
- Execution state

Together,

these provide the Agent with everything it currently knows.

---

# Dynamic Context Workflow

A typical workflow looks like this.

```text
User Request

↓

Create Initial Context

↓

Execute Tool

↓

Update Context

↓

Retrieve Memory

↓

Update Context

↓

Continue Reasoning

↓

Generate Response
```

The context is updated after every important step.

---

# Example

User

```text
Find the cheapest flight to Tokyo.
```

Initial Context

```text
Destination = Tokyo
```

After searching flights

```text
Destination = Tokyo

Cheapest Flight = ₹48,000
```

After checking weather

```text
Destination = Tokyo

Cheapest Flight = ₹48,000

Weather = Rain
```

The Agent's context becomes richer over time.

---

# Dynamic Context vs Static Context

|Static Context|Dynamic Context|
|---|---|
|Fixed during execution|Changes continuously|
|Created once|Updated repeatedly|
|Limited information|Continuously expanding information|
|Best for simple prompts|Essential for AI Agents|

Modern Agentic AI depends heavily on Dynamic Context.

---

# Context Updates

Every important event may trigger a context update.

Example

```text
Tool Finished

↓

Update Context

--------------------

Memory Retrieved

↓

Update Context

--------------------

User Reply

↓

Update Context
```

The Agent always reasons using the latest available context.

---

# Managing Context Growth

Dynamic Context keeps expanding as an Agent works — but it can't grow forever. Several techniques keep it manageable:

### Pruning

Remove information that is no longer relevant to the current step.

```text
Old tool result (no longer needed) → Removed from active context
```

### Summarization

Instead of keeping every raw detail, compress older parts of the context into a shorter summary.

```text
5 tool calls' worth of raw output → 1 short summary paragraph
```

### Prioritization

Not all context is equally important. Agents can score information by relevance to the current step and keep only the highest-priority items in the active window.

```text
High priority:  Current task, latest tool result
Low priority:   Early exploratory steps, resolved sub-tasks
```

---

# Context Window Limits and Overflow

Every LLM has a maximum context window. As Dynamic Context grows, it can eventually exceed this limit.

```text
Context Size > Model's Max Context Window → Overflow
```

Common strategies to handle this:

- Summarize or drop the oldest/lowest-priority context first
- Store full history externally (e.g., in memory or a database) and only pull in relevant pieces
- Periodically compact the context instead of letting it grow unchecked

This connects directly to the next chapter, **Context Routing**, which decides what to keep, remove, or summarize.

---

# Dynamic Context in Multi-Agent Systems

When multiple Agents work together, Dynamic Context can be:

|Shared Context|Isolated Context|
|---|---|
|All Agents read/write the same context|Each Agent keeps its own context|
|Good for tightly collaborating Agents|Good for independent, parallel Agents|
|Risk of conflicting updates|Risk of duplicated or missed information|

### Handling Concurrent Updates

If multiple Agents update shared context at the same time, updates can conflict or overwrite each other.

```text
Agent A writes: weather = "Rain"
Agent B writes: weather = "Sunny"   (at the same time)
```

Frameworks typically handle this with:

- A single coordinator that merges updates in order
- Namespacing each Agent's contributions (e.g., `agentA.weather`, `agentB.weather`)
- Locking or sequencing writes to shared state

---

# Python Example

A simplified example:

```python
context = {
    "destination": "Tokyo"
}

context["weather"] = "Rain"

context["flight_price"] = 48000

print(context)
```

The context grows as the Agent gathers more information.

---

# Real-World Example

Imagine an AI Customer Support Agent.

Initial Context

```text
Customer Name

Issue Description
```

Later

```text
Customer Name

Issue Description

Order History

Refund Status

Previous Tickets
```

As more information becomes available,

the Agent becomes better equipped to solve the problem.

---

# Industry Insight

Modern Agent frameworks continuously update context during execution.

Examples include:

- LangGraph State Updates
- OpenAI Agents SDK Execution Context
- CrewAI Shared Context
- Google ADK Workflow State
- Semantic Kernel Context Variables

Dynamic Context allows Agents to adapt their reasoning as new information becomes available.

---

# Best Practices

Update context only when new information is relevant.

Remove outdated or unnecessary information.

Clearly separate user input from tool outputs and memory.

Keep context organized to improve reasoning quality.

Monitor context size to avoid exceeding model limits.

Summarize or prune before hitting the context window limit, not after.

In multi-agent systems, decide upfront whether context is shared or isolated.

---

# Common Beginner Mistakes

### Mistake 1

Treating context as fixed.

Production Agents continuously update their context.

---

### Mistake 2

Keeping outdated information.

Old or incorrect context can lead to poor decisions.

---

### Mistake 3

Adding every piece of information.

Only relevant information should remain in the active context.

---

### Mistake 4

Confusing Dynamic Context with Long-Term Memory.

Dynamic Context is temporary and task-specific,

while Long-Term Memory persists across tasks.

---

### Mistake 5

Letting context grow unchecked until it overflows the model's window.

Prune, summarize, or prioritize proactively instead of reacting to failures.

---

### Mistake 6

Sharing context across multiple Agents without a conflict-handling strategy.

Concurrent writes without coordination can silently corrupt the context.

---

# Interview Tip

A common interview question is:

> **What is Dynamic Context in Agentic AI?**

A good answer is:

Dynamic Context is the continuously evolving set of information an AI Agent uses while solving a task. It is updated as the Agent receives user input, retrieves memory, executes tools, and collaborates with other Agents. Because it keeps growing, it must be actively pruned, summarized, or prioritized to stay within the model's context window.

---

# Where is this Used?

- LangGraph
- OpenAI Agents SDK
- CrewAI
- Google ADK
- Semantic Kernel
- Enterprise Agent Platforms

---

# Key Takeaways

- Dynamic Context changes throughout Agent execution.
- It includes user input, memory, tool outputs, plans, and execution state.
- Every important event may update the context.
- Context must be pruned, summarized, or prioritized as it grows.
- Context windows are limited, so overflow must be handled proactively.
- In multi-agent systems, context can be shared or isolated, each with trade-offs.
- Good context management improves reasoning quality.
- Dynamic Context is fundamental to modern Agentic AI.

---

