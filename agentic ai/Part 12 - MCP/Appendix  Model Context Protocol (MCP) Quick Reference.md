

This appendix consolidates the MCP chapters into a single reference: glossary, protocol message formats, primitive comparisons, transport details, a minimal end-to-end code example, error codes, and a security checklist.

---

## A.1 — Glossary

|Term|Definition|
|---|---|
|**MCP**|Model Context Protocol — an open standard for connecting AI applications to external tools, data, and systems.|
|**Host**|The AI application the user interacts with (e.g., Claude Desktop, Cursor, VS Code).|
|**Client**|The component inside the Host that manages a connection to one MCP Server.|
|**Server**|A process that exposes Tools, Resources, Prompts, and other capabilities over MCP.|
|**Transport**|The communication channel between Client and Server (stdio or HTTP/SSE).|
|**JSON-RPC**|The message format MCP uses for requests, responses, and notifications.|
|**Capability Negotiation**|The handshake where Client and Server agree on which features are supported.|
|**Tool**|A callable function exposed by a Server that performs an action.|
|**Resource**|Read-only data or content exposed by a Server.|
|**Prompt**|A reusable, parameterized instruction template exposed by a Server.|
|**Sampling**|A Server's request for the Client's LLM to generate a completion.|
|**Root**|A workspace/location boundary the Client shares with a Server.|
|**Elicitation**|A Server's structured request for missing information from the user.|

---

## A.2 — Architecture at a Glance

```text
┌─────────────────────────────┐
│            Host              │   (Claude Desktop, Cursor, VS Code...)
│  ┌─────────────────────────┐ │
│  │        Client           │ │   (1 Client per Server connection)
│  └───────────┬─────────────┘ │
└──────────────┼───────────────┘
               │  JSON-RPC over Transport (stdio / HTTP+SSE)
               ▼
┌─────────────────────────────┐
│           Server             │   (Weather, GitHub, DB, HR, ...)
│  Tools · Resources · Prompts │
│  Sampling · Roots · Elicit.  │
└─────────────────────────────┘
```

A single Host can run multiple Clients simultaneously, each connected to a different Server.

---

## A.3 — Connection Lifecycle

```text
1. Client launches/connects to Server (via transport)
2. initialize        → exchange protocol version + capabilities
3. initialized        → Client confirms readiness
4. Normal operation    → tools/list, resources/list, tools/call, etc.
5. Notifications        → list_changed, progress, log messages (either direction)
6. shutdown/close       → connection terminated cleanly
```

---

## A.4 — JSON-RPC Message Formats

MCP uses **JSON-RPC 2.0** for all communication. There are three message shapes:

**Request** (expects a response)

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_weather",
    "arguments": { "city": "Mumbai" }
  }
}
```

**Response**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{ "type": "text", "text": "The weather in Mumbai is sunny." }]
  }
}
```

**Notification** (no response expected, no `id`)

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/tools/list_changed"
}
```

**Error Response**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid params: 'city' is required"
  }
}
```

---

## A.5 — Common JSON-RPC / MCP Error Codes

|Code|Name|Meaning|
|---|---|---|
|`-32700`|Parse Error|Malformed JSON was sent|
|`-32600`|Invalid Request|Request object is not valid JSON-RPC|
|`-32601`|Method Not Found|Requested method doesn't exist on the Server|
|`-32602`|Invalid Params|Arguments don't match the expected schema|
|`-32603`|Internal Error|Server encountered an unexpected error|
|`isError: true` (in tool result)|Tool-Level Error|The tool ran, but reports a business-logic failure (e.g., invalid email)|

Protocol-level errors (`-327xx`) mean something went wrong with the _call itself_; `isError` in a tool result means the call succeeded but the _outcome_ was a failure — Agents should handle these differently.

---

## A.6 — The Six MCP Primitives, Side by Side

|Primitive|Purpose|Who Initiates|Read/Write|Human Review Typical?|
|---|---|---|---|---|
|**Tool**|Perform an action|Client/Agent calls it|Can read or write|Yes, for destructive actions|
|**Resource**|Provide data|Client reads it|Read-only|No|
|**Prompt**|Reusable instructions|Client/user selects it|N/A|No|
|**Sampling**|Ask Client's LLM to generate text|Server requests it|N/A|Yes, typically|
|**Root**|Define accessible scope|Client shares it|N/A|N/A (advisory)|
|**Elicitation**|Ask user for missing info|Server requests it|N/A|Yes, inherently|

Mental model:

- **Tool** → _Do something._
- **Resource** → _Read something._
- **Prompt** → _Think this way._
- **Sampling** → _Let me borrow your model._
- **Root** → _Here's where you may look._
- **Elicitation** → _I need more info from you._

---

## A.7 — Transports Compared

||stdio|HTTP / SSE|
|---|---|---|
|**Where Server runs**|Local subprocess|Remote/networked|
|**Setup**|`command` + `args`|URL endpoint|
|**Best for**|CLI tools, desktop apps, local dev|Multi-user, cloud-hosted, shared Servers|
|**Auth needed?**|Usually no (same machine)|Usually yes (OAuth, API keys)|

```python
# stdio
server.run(transport="stdio")

# HTTP/SSE
server.run(transport="sse", port=8000)
```

---

## A.8 — SDK Installation Reference

```bash
# Python
pip install mcp

# TypeScript / Node
npm install @modelcontextprotocol/sdk

# Official Inspector (no install needed, run via npx)
npx @modelcontextprotocol/inspector python server.py
```

---

## A.9 — Minimal End-to-End Example (Server + Client)

**server.py**

```python
from mcp.server import Server
from pydantic import BaseModel, Field

server = Server("weather-server")

class WeatherInput(BaseModel):
    city: str = Field(description="City name, e.g. 'Mumbai'")

@server.tool(read_only_hint=True, idempotent_hint=True)
def get_weather(input: WeatherInput):
    """Get current weather conditions for a specific city."""
    return f"The weather in {input.city} is sunny."

@server.resource("docs://company/leave-policy")
def leave_policy():
    return "Employees receive 20 days of annual leave."

@server.prompt()
def summarize_weather(city: str):
    return f"Write a one-sentence, friendly weather summary for {city}."

if __name__ == "__main__":
    server.run(transport="stdio")
```

**client.py**

```python
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

server_params = StdioServerParameters(command="python", args=["server.py"])

async def main():
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            tools = await session.list_tools()
            print("Available tools:", [t.name for t in tools.tools])

            result = await session.call_tool("get_weather", {"city": "Mumbai"})
            print("Result:", result.content[0].text)

asyncio.run(main())
```

**Run it**

```bash
python client.py
```

**Test it interactively (no client code needed)**

```bash
npx @modelcontextprotocol/inspector python server.py
```

---

## A.10 — Bridging MCP Tools into an LLM Agent Loop

```python
mcp_tools = await session.list_tools()

llm_tools = [
    {
        "name": t.name,
        "description": t.description,
        "input_schema": t.inputSchema
    }
    for t in mcp_tools.tools
]

response = llm.invoke(messages, tools=llm_tools)

if response.tool_call:
    result = await session.call_tool(
        response.tool_call.name,
        response.tool_call.arguments
    )
```

This is the piece that connects "MCP exposes capabilities" to "the Agent decides when to use them."

---

## A.11 — Security Checklist for Production MCP

```text
☐ Tools follow least-privilege design (narrow, purpose-built — not raw SQL/shell access)
☐ Destructive Tools are annotated (destructive_hint) and require human confirmation
☐ Roots are backed by real sandboxing / OS-level enforcement, not just advisory scope
☐ Sampling requests use the narrowest includeContext needed
☐ Elicitation is never used to collect passwords, payment info, or other credentials
☐ Authentication/authorization implemented for remote (HTTP/SSE) Servers
☐ Input validated via structured schemas (Pydantic or equivalent) on every Tool
☐ Errors returned as structured isError responses, not raw exceptions
☐ Rate limiting and timeouts applied to Tool calls and Sampling requests
☐ Logging in place for auditing Tool calls, especially destructive ones
```

---

## A.12 — Quick Troubleshooting Reference

|Symptom|Likely Cause|Where to Look|
|---|---|---|
|Client can't connect|Wrong transport/command, Server crashed on start|Run Server directly first; check stdout/stderr|
|Tool not showing up|Not decorated/registered, naming typo|`tools/list` response, MCP Inspector Tools tab|
|Tool call hangs|No timeout set, Server blocked on I/O|Add `asyncio.wait_for`, check async tool implementation|
|"Method not found"|Client/Server capability mismatch|Check `initialize` capability negotiation|
|Elicitation never resolves|No timeout defined|Add a timeout + fallback (Chapter 79)|
|Stale tool list after workspace change|Not listening for `list_changed`|Handle `roots/list_changed` / `tools/list_changed` notifications|

---

## A.13 — Further Reading

- Official MCP specification and docs: https://modelcontextprotocol.io
- MCP Inspector: `npx @modelcontextprotocol/inspector`
- Python SDK: `pip install mcp`
- TypeScript SDK: `npm install @modelcontextprotocol/sdk`

---

_This appendix summarizes Chapters 68–87 (MCP Fundamentals through Testing with MCP Inspector) for quick lookup. Refer back to the individual chapters for full explanations, diagrams, and worked examples._


