

## Learning Objectives

By the end of this chapter, you will understand:

- What Context Routing is
- Why Context Routing is important
- How AI Agents decide which context to use
- Different Context Routing strategies
- How to evaluate routing quality
- How permissions affect what context is routed
- How to handle routing failures
- Advantages and challenges of Context Routing
- Best practices for Context Routing

---

# Introduction

Imagine you ask an AI Agent:

```text
Book me the cheapest flight to Tokyo.
```

The Agent has access to:

- Previous conversations
- User preferences
- Weather information
- Flight search results
- Hotel recommendations
- Company policies
- Travel history

Should the Agent send **all** of this information to the LLM?

No.

Much of it is irrelevant.

Instead,

the Agent selects only the information needed for the current task.

This process is called **Context Routing**.

---

# What is Context Routing?

Context Routing is the process of selecting the most relevant information before sending it to an AI model.

Instead of providing every available piece of context,

the Agent routes only the information required for the current reasoning step.

This improves both efficiency and response quality.

---

# Why is Context Routing Important?

Modern AI Agents collect information from many sources.

Examples include:

- User conversations
- Memory
- Tool outputs
- Other Agents
- External databases
- Documents

Not all of this information is useful for every task.

Sending unnecessary context can:

- Increase cost
- Increase latency
- Confuse the LLM
- Waste context window space

Context Routing solves this problem.

---

# Visual Diagram

```text
          Available Context

 ┌─────────────────────────────┐
 │ User History                │
 │ Tool Results                │
 │ Memory                      │
 │ Documents                   │
 │ Other Agent Outputs         │
 └─────────────────────────────┘

               │

        Context Routing

               │

               ▼

      Relevant Context Only

               │

               ▼

              LLM
```

The Agent filters information before sending it to the model.

---

# Context Routing Workflow

A typical workflow looks like this.

```text
Collect Context

↓

Evaluate Relevance

↓

Select Important Information

↓

Build Prompt

↓

Send to LLM

↓

Generate Response
```

Only useful context reaches the model.

---

# Sources of Context

An Agent may gather context from multiple places.

Examples include:

- Current user request
- Conversation history
- Retrieved memories
- Tool outputs
- Shared Memory
- Vector Database
- External APIs
- Other Agents

Context Routing determines which of these sources should be included.

---

# Routing Strategies

Different systems use different routing strategies.

---

## 1. Rule-Based Routing

The Agent follows predefined rules.

Example

```text
Travel Question

↓

Include Flights

Include Hotels

Ignore Medical History
```

Simple,

but less flexible.

---

## 2. Similarity-Based Routing

The Agent retrieves context that is most similar to the current request.

Example

```text
User Query

↓

Vector Search

↓

Top Relevant Documents
```

This is commonly used in RAG systems.

---

## 3. Priority-Based Routing

Some information is always considered more important.

Example

Priority

```text
Current User Request

↓

Current Task

↓

Recent Tool Results

↓

Conversation History
```

Higher-priority context is included first.

---

## 4. LLM-Based Routing

The LLM itself helps determine which context is relevant.

Example

```text
Available Context

↓

LLM Evaluation

↓

Relevant Context Selected
```

Useful for complex reasoning tasks.

---

## 5. Hybrid Routing

Many production systems combine:

- Rules
- Similarity Search
- Priority
- LLM Reasoning

This provides both speed and accuracy.

---

# Comparing Routing Strategies

|Strategy|Speed|Flexibility|Best For|
|---|---|---|---|
|Rule-Based|Fast|Low|Predictable, well-defined tasks|
|Similarity-Based|Medium|Medium|RAG, document/knowledge retrieval|
|Priority-Based|Fast|Medium|Ensuring critical context is never dropped|
|LLM-Based|Slow|High|Complex, ambiguous reasoning tasks|
|Hybrid|Medium|High|Production systems needing both speed and accuracy|

There's no single "best" strategy — the right choice depends on the task's complexity and latency/cost budget.

---

# Example

User

```text
What hotels do you recommend in Tokyo?
```

Available Context

```text
Flight Details

Hotel Search Results

Weather

Python Code

Employee Records
```

Context Routing selects:

```text
Hotel Search Results

Weather
```

The remaining information is excluded.

---

# Python Example

A simplified example:

```python
available_context = [
    "Hotel Results",
    "Weather",
    "Employee Records"
]

selected_context = [
    item for item in available_context
    if item != "Employee Records"
]

print(selected_context)
```

Production systems use semantic search,

ranking,

and intelligent filtering,

but the principle is the same.

---

# Access-Control / Permission-Based Routing

Context Routing isn't only about relevance — it's also about **security**.

Some context should never be routed to certain Agents or users, regardless of how relevant it seems.

```text
User asks: "What is my colleague's salary?"

↓

Relevant context exists (Payroll Records)

↓

But: Permission check fails → Excluded from routing
```

Best practices:

- Tag context sources with access levels (e.g., public, internal, restricted)
- Check the requesting user/Agent's permissions before including sensitive context
- Log when sensitive context is excluded, for auditing

This prevents relevance-based routing from accidentally leaking data the requester isn't authorized to see.

---

# Evaluating Routing Quality

Like prompts, routing decisions should be measured, not assumed to be correct.

Common checks:

- **Precision**: Of the context routed, how much was actually useful?
- **Recall**: Of the context that _should_ have been included, how much was actually routed?
- **Downstream accuracy**: Did the final response improve when routing was correct?

```text
Routed 5 items → 4 were relevant → Precision = 80%
5 relevant items existed → 4 were routed → Recall = 80%
```

Teams often build a small evaluation set (similar to the golden dataset from Chapter 35) specifically to test routing decisions.

---

# Handling Routing Failures

Sometimes no context is found to be relevant, or the routing step makes a bad choice.

```text
No relevant context found
↓
Options:
- Ask the user a clarifying question
- Fall back to a broader search
- Proceed with a general-purpose response, flagged as low-confidence
```

A robust system should have a defined fallback behavior instead of silently sending an empty or incorrect context to the LLM.

---

# Routing Overhead

Context Routing itself isn't free — similarity search, LLM-based evaluation, or permission checks all add some latency and cost.

```text
Total Time = Routing Time + LLM Response Time
```

For simple tasks, lightweight rule-based or priority-based routing is often enough — reserving expensive LLM-based routing for genuinely complex cases keeps overhead in check.

---

# Real-World Example

Imagine an Enterprise HR Assistant.

User

```text
How many leave days do I have remaining?
```

Available Context

```text
Payroll Records

Leave Balance

IT Tickets

Travel Requests

Training Courses
```

Context Routing selects:

```text
Leave Balance
```

Everything else is ignored.

This reduces cost and improves accuracy.

---

# Industry Insight

Context Routing is a core component of modern Agentic AI.

Examples include:

- LangGraph context management
- OpenAI Agents SDK execution context
- Google ADK workflow context
- CrewAI shared context selection
- RAG pipelines using vector search

Most enterprise AI systems perform Context Routing before every reasoning step.

---

# Best Practices

Select only information relevant to the current task.

Prioritize recent and task-specific information.

Remove duplicate context.

Continuously update routing decisions as the task evolves.

Measure routing quality during testing.

Enforce access control before relevance filtering, not after.

Define a fallback behavior for when no relevant context is found.

---

# Common Beginner Mistakes

### Mistake 1

Sending all available context to the LLM.

More context does not always produce better results.

---

### Mistake 2

Ignoring relevance.

Only information that helps solve the current task should be included.

---

### Mistake 3

Using static routing rules for every request.

Different tasks require different context.

---

### Mistake 4

Not updating context after new tool results or memory retrievals.

Context Routing should adapt throughout execution.

---

### Mistake 5

Routing purely by relevance and ignoring permissions.

Sensitive context can leak to the wrong user or Agent if access control isn't checked first.

---

### Mistake 6

Having no fallback when routing finds nothing relevant.

Silently proceeding with empty or poor context leads to low-quality, unpredictable answers.

---

# Interview Tip

A common interview question is:

> **What is Context Routing in Agentic AI?**

A good answer is:

Context Routing is the process of selecting the most relevant information from multiple context sources before sending it to an AI model. It improves reasoning quality, reduces cost, lowers latency, and makes better use of the model's context window — while also enforcing access control so sensitive information isn't routed to unauthorized users or Agents.

---

# Where is this Used?

- LangGraph
- OpenAI Agents SDK
- CrewAI
- Google ADK
- RAG Systems
- Enterprise AI Platforms

---

# Key Takeaways

- Context Routing selects the most relevant information for the current task.
- It prevents unnecessary information from reaching the LLM.
- Common routing strategies include rule-based, similarity-based, priority-based, LLM-based, and hybrid routing.
- Routing quality can be measured using precision, recall, and downstream accuracy.
- Permission-based routing protects sensitive context from unauthorized access.
- Robust systems define fallback behavior for when routing finds nothing relevant.
- Routing itself has a cost/latency overhead that should be balanced against task complexity.
- Good Context Routing improves accuracy, efficiency, and scalability.
- Modern Agentic AI systems perform Context Routing continuously during execution.

---

