

## Learning Objectives

By the end of this chapter, you will understand:

- What Context Summarization is
- Why AI Agents summarize context
- Different Context Summarization techniques
- When to trigger summarization
- How to measure summarization quality
- When to summarize instead of filter
- Advantages and limitations of summarization
- Best practices for Context Summarization

---

# Introduction

Imagine an AI Agent helping a user over several days.

The conversation now contains:

- 500 messages
- Multiple tool calls
- Retrieved documents
- Planning steps
- Intermediate results

The Agent cannot send all this information to the LLM every time.

Deleting everything isn't a good idea either,

because important information may be lost.

Instead,

the Agent compresses the important information into a shorter version.

This process is called **Context Summarization**.

---

# What is Context Summarization?

Context Summarization is the process of compressing large amounts of context into a shorter version while preserving the important information.

Instead of removing information,

the Agent rewrites it in a more compact form.

The goal is to reduce token usage without losing essential meaning.

---

# Why is Context Summarization Important?

As conversations and workflows grow,

the context becomes larger.

Without summarization,

the Agent may:

- Exceed the context window
- Increase API costs
- Slow down reasoning
- Lose important historical information

Summarization allows the Agent to retain knowledge while using fewer tokens.

---

# Visual Diagram

```text
Large Context

↓

Context Summarization

↓

Compact Summary

↓

LLM
```

The Agent keeps the important information while reducing size.

---

# Context Summarization Workflow

A typical workflow looks like this.

```text
Collect Context

↓

Identify Important Information

↓

Generate Summary

↓

Replace Original Context

↓

Send Summary to LLM
```

The summarized version becomes part of the new context.

---

# When Should Agents Summarize?

Summarization is useful when:

- Conversations become very long.
- Tool outputs are too large.
- Retrieved documents exceed token limits.
- Execution history grows continuously.
- Context filtering alone is insufficient.

---

# When to Trigger Summarization

Summarization itself costs an extra LLM call, so it shouldn't run on every single turn. Common triggers include:

### Token Threshold

```text
If conversation tokens > 70% of context window
→ Trigger summarization
```

### Message Count

```text
Every 50 messages → Summarize the oldest 40, keep the last 10 raw
```

### Idle/Checkpoint-Based

```text
At the end of a completed task or sub-task → Summarize that segment
```

Choosing a clear trigger avoids summarizing too often (wasting cost) or too rarely (risking overflow).

---

# The Cost of Summarization

Summarization isn't free — generating a summary is itself an LLM call, which adds latency and token cost.

```text
Total Cost = Cost of Summarization Call + Cost of Main Response Call
```

This trade-off is usually worth it because:

- The summary is reused across many future turns, amortizing its one-time cost
- It prevents far more expensive overflow/failure scenarios later

Still, teams should track how often summarization runs and its cost, the same way they track any other LLM call (Chapter 38).

---

# Summarization Techniques

## 1. Conversation Summarization

Compress previous conversations.

Example

Before

```text
200 Messages
```

After

```text
Customer prefers premium hotels and has already rejected three flight options.
```

---

## 2. Document Summarization

Compress long documents.

Example

```text
100 Pages

↓

2-Page Summary
```

---

## 3. Execution Summarization

Summarize completed workflow steps.

Example

Before

```text
Step 1 Completed

Step 2 Completed

Step 3 Completed

...
```

After

```text
Data collection and preprocessing completed successfully.
```

---

## 4. Hierarchical Summarization

Summarize small sections first,

then summarize those summaries.

```text
Document A

↓

Summary A

--------------------

Document B

↓

Summary B

↓

Final Summary
```

Useful for very large datasets.

---

## 5. Rolling Summarization

Continuously replace old conversations with updated summaries.

```text
Messages

↓

Summary

↓

New Messages

↓

Updated Summary
```

Many production chat systems use this approach.

---

## 6. Structured Summarization

Instead of compressing into free-flowing prose, extract key facts into a structured format (like key-value pairs or JSON).

Example

Prose summary

```text
The customer wants a premium hotel in Tokyo under ₹15,000/night and has already rejected two economy flights.
```

Structured summary

```json
{
  "destination": "Tokyo",
  "hotel_preference": "premium",
  "budget_per_night": 15000,
  "rejected_flights": 2
}
```

Structured summaries are less likely to lose or blur specific facts than prose, and they're easier for downstream code to check or update precisely.

---

# Summarization vs Filtering

These techniques work together,

but they solve different problems.

|Context Filtering|Context Summarization|
|---|---|
|Removes information|Compresses information|
|Deletes irrelevant content|Preserves important content|
|Reduces size by omission|Reduces size by rewriting|

Production Agents often perform filtering first,

then summarize what remains.

---

# Summary Quality

A good summary should be:

- Accurate
- Complete
- Concise
- Relevant
- Easy to understand

A poor summary may lose critical information,

leading to incorrect reasoning.

---

# Measuring Summarization Quality (Faithfulness)

Because summaries are generated by an LLM, they can introduce errors — this is called a **faithfulness** problem: the summary says something the original content didn't actually say.

```text
Original: "The client is undecided about the Tokyo trip."
Bad Summary: "The client confirmed the Tokyo trip."   ← Unfaithful
```

Ways to check summary quality:

- Spot-check summaries against the original content on a sample basis
- Use an LLM-as-judge (Chapter 35) to score whether the summary is faithful to the source
- Track downstream task accuracy before and after introducing summarization

A summary that saves tokens but introduces incorrect facts can be worse than no summarization at all.

---

# Archiving Original Content

Just like with filtering (Chapter 51), the original, unsummarized content shouldn't simply be discarded once a summary is created.

```text
Original Content
↓
Summarized for Active Context
↓
Original still stored in Memory/Logs for later retrieval
```

If the Agent (or a human reviewer) later needs the exact original wording — for example, in a legal or compliance context — it should still be retrievable, even though the active context only holds the compressed version.

---

# Python Example

A simplified example:

```python
conversation = load_chat_history()

summary = llm.summarize(conversation)

context = summary

response = llm.invoke(context)
```

Production systems often summarize only older conversations while keeping recent messages unchanged.

---

# Real-World Example

Imagine an AI Legal Assistant.

Available Context

```text
300 Pages of Contracts

100 Chat Messages

Legal Research

Case History
```

The Agent:

- Filters irrelevant documents.
- Summarizes long contracts.
- Summarizes old conversations.
- Keeps recent legal instructions unchanged.

This allows the Agent to remain within the context window.

---

# Industry Insight

Context Summarization is widely used in production Agent systems.

Examples include:

- LangGraph conversation summaries
- OpenAI Agents SDK memory compression
- CrewAI long-running workflows
- Google ADK context optimization
- Enterprise chatbot platforms

Long-running AI Agents often summarize history automatically to maintain performance.

---

# Best Practices

Summarize only when necessary.

Keep recent conversations unchanged whenever possible.

Preserve important facts and decisions.

Review summary quality during testing.

Regenerate summaries if important information is lost.

Use a clear trigger (token threshold, message count, or checkpoint) instead of summarizing on every turn.

Prefer structured summaries for critical facts that must stay precise.

Archive the original content even after summarizing it.

---

# Common Beginner Mistakes

### Mistake 1

Summarizing too aggressively.

Important details may disappear.

---

### Mistake 2

Replacing recent context with summaries.

Recent interactions often contain the most relevant information.

---

### Mistake 3

Assuming summaries are always accurate.

LLMs may omit or alter important facts.

---

### Mistake 4

Using summaries instead of retrieval.

Sometimes retrieving the original document is better than relying on a summary.

---

### Mistake 5

Summarizing on every single turn regardless of need.

This adds unnecessary LLM calls, cost, and latency — use a clear trigger instead.

---

### Mistake 6

Discarding the original content after summarizing.

If exact wording is needed later, an unfaithful or lossy summary can't be corrected without the source.

---

# Interview Tip

A common interview question is:

> **What is Context Summarization?**

A good answer is:

Context Summarization is the process of compressing large amounts of context into a shorter version while preserving important information. It helps AI Agents stay within context window limits while maintaining enough information for accurate reasoning — and its quality should be measured for faithfulness, since a summary generated by an LLM can introduce errors.

---

# Where is this Used?

- LangGraph
- OpenAI Agents SDK
- CrewAI
- Google ADK
- Enterprise AI Platforms
- Long-Running AI Agents

---

# Key Takeaways

- Context Summarization compresses information instead of removing it.
- It helps AI Agents stay within context window limits.
- Common techniques include conversation, document, execution, hierarchical, rolling, and structured summarization.
- Summarization should be triggered by a clear rule (token threshold, message count, or checkpoint), not on every turn.
- Summarization itself has a cost — an extra LLM call — that should be tracked.
- Summary faithfulness should be measured; an inaccurate summary can hurt more than help.
- Original content should be archived even after it's summarized.
- Filtering removes information, while summarization preserves important information in a shorter form.
- Production Agent systems frequently summarize long-running conversations and workflows.

---


