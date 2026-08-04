

## Learning Objectives

By the end of this chapter, you will understand:

- What a Context Window is
- Why Context Window Management is important
- How tokens are actually counted
- What happens when the context window is exceeded
- How context window sizes vary across models
- Common Context Window Management techniques
- Token budgeting
- Prompt caching for repeated context
- Best practices for managing large contexts

---

# Introduction

Imagine an AI Agent is helping a user with a large software project.

During execution, it collects:

- User instructions
- Conversation history
- Retrieved documents
- Tool outputs
- Memory
- Code files
- Planning results

Eventually, the Agent has **50,000 tokens** of information.

However, the LLM can only process **16,000 tokens**.

The Agent cannot send everything.

It must decide:

- What should be included?
- What should be removed?
- What should be summarized?

Managing this limited space is called **Context Window Management**.

---

# What is a Context Window?

A Context Window is the maximum amount of information (measured in **tokens**) that an LLM can process in a single request.

It includes everything sent to the model, such as:

- System Prompt
- User Prompt
- Conversation History
- Retrieved Documents
- Tool Results
- Agent State
- Memory
- Previous Responses

If the total exceeds the model's limit,

some information must be removed or compressed.

---

# How Are Tokens Counted?

A token is not the same as a word or character — it's a chunk of text the model's tokenizer breaks input into.

```text
"ChatGPT is helpful" → ["Chat", "G", "PT", " is", " helpful"]
```

Rough rule of thumb for English text:

```text
1 token ≈ 4 characters ≈ 0.75 words
```

In practice, teams use the actual tokenizer library for their model (e.g., `tiktoken` for OpenAI models) to get an exact count, rather than estimating.

```python
import tiktoken

encoding = tiktoken.encoding_for_model("gpt-4")
tokens = encoding.encode("ChatGPT is helpful")
print(len(tokens))
```

Estimating with rules of thumb is fine for rough budgeting, but exact counts matter when you're close to the limit.

---

# Context Window Sizes Across Models

Context window limits vary significantly between models and change frequently as providers release updates.

|Model Family|Typical Context Window (approximate)|
|---|---|
|Older GPT-3.5 style models|~4K–16K tokens|
|GPT-4 class models|~32K–128K tokens|
|Claude models|~100K–200K+ tokens|
|Gemini models|~1M+ tokens (largest variants)|

> Exact numbers change often — always check the current provider documentation rather than relying on memorized figures when building a production system.

A larger context window gives more room but doesn't remove the need for good context management — cost and latency still scale with tokens used.

---

# Why is Context Window Management Important?

Large Agent systems continuously collect information.

Without proper management:

- Important information may be lost.
- API costs increase.
- Response time increases.
- The model may ignore relevant context.
- Requests may exceed the model's limit.

Good Context Window Management ensures the model receives the most useful information.

---

# Visual Diagram

```text
Available Context

↓

System Prompt

↓

Conversation

↓

Memory

↓

Tool Results

↓

Documents

↓

Agent State

↓

Context Window Management

↓

LLM
```

Only the information that fits within the context window is sent to the model.

---

# What Uses the Context Window?

Everything consumes tokens.

Typical consumers include:

- System Instructions
- User Messages
- Conversation History
- Retrieved Documents
- Tool Outputs
- Agent Plans
- Memory
- Previous Responses

Agents must carefully manage each of these components.

---

# What Happens When the Context Window is Full?

Suppose an LLM supports:

```text
128,000 Tokens
```

The Agent builds a prompt containing:

```text
145,000 Tokens
```

The request cannot be processed as-is.

The Agent must:

- Remove unnecessary information
- Summarize content
- Retrieve only relevant data
- Reduce conversation history

before sending the request.

---

# Handling Overflow Errors Gracefully

If a request does exceed the limit, the API call itself will typically fail with an error rather than silently truncating.

```python
try:
    response = llm.invoke(context)
except ContextLengthExceededError:
    context = summarize(context)
    response = llm.invoke(context)
```

Production systems should:

- Catch this error explicitly rather than letting the request crash
- Trigger a fallback (summarize, trim, or retrieve less) and retry
- Log how often overflow happens — frequent overflow signals the token budget needs adjusting upstream

Relying on the error as the first line of defense is risky; proactive measurement (checking token count before sending) is safer and faster than reacting to failures.

---

# Context Window Management Workflow

A typical workflow looks like this.

```text
Collect Context

↓

Measure Tokens

↓

Prioritize Context

↓

Filter / Summarize

↓

Fit Within Token Budget

↓

Send to LLM
```

This process happens before every model call.

---

# Common Techniques

## 1. Context Filtering

Remove irrelevant information.

Example

```text
Old Conversation

↓

Remove
```

---

## 2. Context Summarization

Compress large amounts of information.

Example

```text
20 Pages

↓

1-Page Summary
```

---

## 3. Retrieval

Instead of sending everything,

retrieve only what is needed.

Example

```text
Knowledge Base

↓

Top 5 Relevant Documents
```

---

## 4. Sliding Window

Keep only the most recent conversation.

Example

```text
Messages 1–100

↓

Keep Messages 91–100
```

Older messages are discarded or summarized.

---

## 5. Token Budgeting

Reserve tokens for different types of information.

Example

```text
System Prompt

2,000 Tokens

Conversation

5,000 Tokens

Memory

3,000 Tokens

Tool Outputs

2,000 Tokens

Response

4,000 Tokens
```

This prevents one section from consuming the entire context window.

---

# Token Budgeting

Production Agents often allocate a token budget.

Example

```text
Context Window

128,000 Tokens

↓

System Prompt

5%

↓

Conversation

30%

↓

Memory

20%

↓

Documents

30%

↓

Response

15%
```

This ensures balanced use of the available context.

---

# Prompt Caching

Some LLM providers support **prompt caching**: if the beginning portion of a prompt (like a long system prompt or a large document) stays identical across requests, the provider can reuse its internal processing instead of recomputing it every time.

```text
Request 1: [Long System Prompt + Docs] + Question A → Full processing
Request 2: [Same Long System Prompt + Docs] + Question B → Cached prefix reused
```

Benefits:

- Lower latency for repeated large context
- Reduced cost on the cached portion

To take advantage of this, keep stable content (system instructions, reference documents) at the start of the prompt, and put frequently changing content (the user's latest question) at the end.

---

# Long Context vs Retrieval (RAG)

With very large context windows now available, a natural question is: why not just put everything in the prompt instead of using retrieval?

|Long Context|Retrieval (RAG)|
|---|---|
|Simpler to implement|Requires a retrieval/search step|
|Cost and latency scale with data size|Only relevant chunks are sent, keeping cost lower|
|Can struggle to focus on the right details in huge inputs|Explicitly narrows down to what matters|
|Good for moderate, bounded data|Good for very large or constantly growing knowledge bases|

In practice, most production systems combine both: retrieval narrows down candidates, and a reasonably sized context window holds the selected results.

---

# Context Window vs Context Construction

These concepts are related but different.

|Context Construction|Context Window Management|
|---|---|
|Builds the final context|Ensures it fits within the token limit|
|Decides what to include|Decides how much can be included|
|Focuses on relevance|Focuses on size and token limits|

Production systems perform both steps before every LLM request.

---

# Python Example

A simplified example:

```python
MAX_TOKENS = 8000

context = get_context()

if count_tokens(context) > MAX_TOKENS:
    context = summarize(context)

response = llm.invoke(context)
```

Real systems use tokenizer libraries to measure token counts accurately.

---

# Real-World Example

Imagine an AI Coding Assistant.

Available Context

```text
System Instructions

Conversation History

50 Source Files

Documentation

Tool Results
```

The Agent:

- Retrieves only relevant files.
- Summarizes long conversations.
- Removes completed tasks.
- Allocates a token budget.

The final prompt fits within the model's context window.

---

# Industry Insight

Context Window Management is a core component of every production Agent framework.

Examples include:

- LangGraph Context Management
- OpenAI Agents SDK
- Google ADK
- CrewAI
- Semantic Kernel

Modern enterprise systems continuously measure and optimize token usage before every LLM call.

---

# Best Practices

Measure token usage before sending requests.

Reserve space for the model's response.

Retrieve only relevant information.

Summarize long conversations.

Avoid sending duplicate context.

Continuously monitor token consumption.

Place stable content early in the prompt to take advantage of prompt caching.

Handle context-length-exceeded errors with a defined fallback, not a crash.

---

# Common Beginner Mistakes

### Mistake 1

Assuming more context always produces better answers.

Irrelevant context can reduce response quality.

---

### Mistake 2

Ignoring token limits.

Requests that exceed the model's context window may fail or require truncation.

---

### Mistake 3

Using the entire context window for input.

Always reserve tokens for the model's response.

---

### Mistake 4

Sending duplicate information.

Repeated content wastes valuable context space.

---

### Mistake 5

Estimating token count instead of using the actual tokenizer.

Rough word-based estimates can be wrong enough to cause unexpected overflow near the limit.

---

### Mistake 6

Reaching for a larger context window instead of proper retrieval.

A bigger window doesn't replace good filtering and retrieval — cost and quality still suffer if everything is dumped in.

---

# Interview Tip

A common interview question is:

> **What is Context Window Management?**

A good answer is:

Context Window Management is the process of ensuring that all information sent to an LLM fits within its maximum token limit. It involves measuring token usage with the model's tokenizer, filtering irrelevant information, summarizing large content, retrieving only relevant data, allocating token budgets efficiently, and handling overflow errors gracefully.

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

- Every LLM has a limited context window measured in tokens.
- Tokens are measured with a tokenizer, not simple word counts — use exact libraries near the limit.
- Context window sizes vary widely across models and change often; always check current docs.
- AI Agents must manage what information fits into that window.
- Common techniques include filtering, summarization, retrieval, sliding windows, and token budgeting.
- Prompt caching can reduce cost and latency for repeated stable context.
- Long context windows and retrieval (RAG) are complementary, not mutually exclusive.
- Good Context Window Management improves performance, reduces cost, and prevents token limit errors.
- Production Agent systems optimize the context window before every model invocation.

---

