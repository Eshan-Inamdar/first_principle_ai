---
title: "Agentic AI Deep Dive - Tools: How Agents Act on the World"
seoTitle: "AI Agent Tools Explained — And What MCP Changes"
seoDescription: " Learn how tool calls actually work, what makes a tool
reliable, and why MCP is standardizing the entire
agent tool ecosystem"
datePublished: 2026-05-18T15:49:20.481Z
cuid: cmpbdqcbh001y2gjze5pzdvqm
slug: agentic-ai-deep-dive-tools-how-agents-act-on-the-world
cover: https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/b8dd1c94-ce12-49f5-94ff-b522e32bec36.png
tags: ai, artificial-intelligence, data-science, machine-learning, machinelearning, ai-tools, llm, aitools, ai-agents, artificialintelligence, AI

---

* * *

## Recap: The Story So Far

Three parts in. Here's where we stand.

In **Part 1** we established why LLMs alone break for real work. Four fundamental limitations — knowledge cutoff, no private data, statelessness, no actions. RAG fixed the knowledge problem. But it didn't give the LLM hands. The insight that changed everything: an LLM can produce a *next action* rather than a final answer, and operate inside a reasoning loop.

In **Part 2** we opened the hood. Every agent has five core components — LLM, planner, executor, tools, and memory — held together by a harness that manages the loop. The LLM is the decision-maker, not the whole system.

In **Part 3** we explored how agents think. ReAct — reason before every action, observe the result, repeat. Plan-and-Execute for longer structured tasks. Dynamic replanning when steps break mid-execution.

All of that thinking has been building to one moment: the agent decides to *do something*.

That's what tools are for. And without them, everything we've covered so far produces nothing but text.

* * *

## The Hook

A surgeon with no instruments is just a person with knowledge.

The knowledge matters — enormously. But knowledge alone doesn't remove a tumor, set a bone, or close a wound. The instruments are what translate expertise into action. And the quality of those instruments — how precise they are, how reliable, how well-suited to the task — determines what's actually possible in the operating room.

An agent's tools work exactly the same way. The LLM is the expertise. The tools are the instruments. And just like in surgery, a poorly designed tool in a critical moment doesn't just fail — it can make things worse.

* * *

## What a Tool Call Actually Is

Here's something most people get wrong about agents.

When we say an agent "calls a tool," it sounds like the LLM is executing code. Reaching out to an API. Running a function. But that's not what's happening.

The LLM produces text. That's all it ever does.

What actually happens when an agent calls a tool:

```plaintext
1. LLM produces structured text describing an action:
   { "tool": "web_search", "query": "MCP protocol 2025" }

2. The harness intercepts that output

3. The harness runs the actual function — the real API call,
   the real database query, the real file operation

4. The result is formatted and returned to the LLM
   as an observation in the next loop iteration
```

The LLM never touches the external world directly. It describes what it wants to do. The harness does it. The result comes back as text the LLM can reason about.

This distinction matters more than it seems. It means the LLM's ability to use a tool is entirely dependent on how well the tool is *described* to it. The LLM has no other way to know what a tool does, when to use it, or what to pass it — except through the description you write.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/9f6d3cac-0d72-4003-82b3-e65557b05657.png align="center")

* * *

## The Three Parts of Every Tool

Every tool an agent can use has exactly three things:

**Name** — what the tool is called. Should be clear and unambiguous. `search_web` is better than `search`. `query_database` is better than `db`. The name is the first signal the LLM uses to decide whether a tool is relevant to the current step.

**Description** — what the tool does, when to use it, and when *not* to use it. This is the most important part and the most commonly underinvested. A good description answers: what does this tool return? What are its limitations? When should I reach for this over another tool?

**Schema** — the parameters the tool accepts. Names, types, whether they're required or optional, what valid values look like. The LLM uses the schema to construct the tool call. A poorly designed schema produces malformed calls.

Here's the difference between a bad tool definition and a good one:

**Bad:**

```plaintext
Name: search
Description: searches the internet
Parameters: query (string)
```

**Good:**

```plaintext
Name: web_search
Description: Searches the web for current information.
             Use when you need facts, news, or data that
             may have changed recently. Returns the top 5
             results with titles, URLs, and snippets.
             Do NOT use for internal company data —
             use query_database instead.
Parameters:
  query (string, required): the search query, 3-8 words
  max_results (integer, optional): default 5, max 10
```

The second version tells the LLM not just what the tool does but *when to use it* and *when not to*. That context is what prevents the tool from being called with wrong parameters — or called when it shouldn't be called at all.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/d7d2b3cf-9a47-45c7-b653-3841b7225ac8.png align="center")

* * *

## Tool Design Principles

Good tool design is an underrated engineering skill. Here are the principles that separate reliable tools from fragile ones.

**One tool, one job.** A tool that does three things creates ambiguity. The LLM has to guess which of its behaviors is relevant right now. Split multi-purpose tools into single-purpose ones. `read_file` and `write_file` are better than `file_operations`.

**Fail loudly, never silently.** A tool that returns an empty result when it fails looks identical to a tool that found nothing. The LLM has no way to tell the difference. Design tools to return explicit error states. `{"error": "rate_limit_exceeded", "retry_after": 30}` is far better than `{}`.

**Return what the LLM needs, not everything.** A tool that returns a 50,000-word document when the LLM asked a yes/no question wastes context window and forces the model to find the signal in noise. Pre-process and filter tool outputs before they reach the LLM.

**Validate inputs before executing.** Check parameter types, ranges, and formats at the tool level before running anything expensive or irreversible. A tool call with a malformed parameter should fail immediately with a clear error — not halfway through an operation.

**Design for retries.** Tools will fail. APIs go down. Timeouts happen. Build tools to be safely retryable — and make it clear in the description whether a tool is idempotent or not.

* * *

## Categories of Tools

Agents use tools that fall into five broad categories. Understanding the categories helps you think about what capabilities your agent needs before you start building.

**Read tools** — retrieve information from the world without changing anything. Web search, database queries, file reads, API GET requests. These are the safest tools to give an agent because they have no side effects.

**Write tools** — create or modify things in the world. File writes, database inserts and updates, API POST/PUT requests, calendar events, emails sent. These need more careful design — a write that goes wrong is harder to undo than a read.

**Execute tools** — run processes or trigger workflows. Code interpreters, shell commands, test runners, deployment scripts. The most powerful category and the most dangerous. Scope them tightly.

**Communicate tools** — send messages through external channels. Slack, email, SMS, webhook triggers. These interact with people, not just systems — which means mistakes have human consequences.

**Computer use tools** — control a browser or desktop UI directly. Click buttons, fill forms, navigate pages, take screenshots. A relatively new category, pioneered by Anthropic's Computer Use capability. Useful when no API exists — you interact with the UI the way a human would.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/29d16fd1-3358-471a-aef7-176ace33e43a.png align="center")

* * *

## MCP — The Standard That Changes Everything

For most of the short history of agentic AI, every framework had its own way of connecting agents to tools.

LangChain had its tool format. AutoGen had its own. CrewAI had its own. If you built a tool for one framework, it didn't work in another. If you wanted to give your agent access to a new service — say, your company's internal database — you had to write a custom integration for whatever framework you were using. Then write it again when you switched frameworks. Then maintain it when either side changed.

This fragmentation was a real problem. Not a theoretical one. It slowed down development, created duplicated work, and made tool ecosystems siloed.

**MCP — Model Context Protocol** — is Anthropic's answer to this problem.

The idea is straightforward: define a standard protocol for how agents communicate with tools and external services. Any tool built to the MCP standard works with any agent built to the MCP standard. You write the integration once. It works everywhere.

Think of it like USB. Before USB, every device had its own connector. Printers, keyboards, mouse, cameras — all different plugs, all different cables, all requiring specific ports. USB standardized the interface. One connector type. Works with everything.

MCP does the same for agent tools.

**How MCP works:**

An **MCP server** exposes tools and resources through a standard interface. It could be a local process, a remote service, or anything in between. It declares what it can do — what tools it offers, what parameters they take, what they return.

An **MCP client** — the agent — connects to MCP servers and gets a catalogue of available tools. It can then call those tools through the standard protocol without knowing anything about the underlying implementation.

```plaintext
Agent (MCP client)
     ↕  standard protocol
MCP Server A — web search tools
MCP Server B — database tools  
MCP Server C — your company's internal API
MCP Server D — calendar and email tools
```

**What this means in practice:**

The ecosystem of MCP servers is growing fast. Slack, GitHub, Google Drive, Postgres, Brave Search — all have MCP servers already. You connect your agent to the servers it needs. The tools are immediately available. No custom integration code.

And when you switch agent frameworks — from LangChain to LangGraph, from a custom harness to Claude Code — your MCP tool connections come with you. The tools aren't tied to the framework. They're tied to the protocol.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/582dc8d4-959d-4bdc-91c1-32fab4b8a26d.png align="center")

* * *

## Tool Failure Modes

Tools fail. This is not an edge case — it's a routine reality in production. The question is how your agent handles it.

**Silent failures** are the most dangerous. A tool returns an empty response or a malformed result without signaling that anything went wrong. The LLM sees the empty result, assumes there's nothing to find, and moves on — producing a wrong answer with full confidence.

**Timeout cascades** happen when one slow tool call delays the entire loop. If your agent has no timeout handling, a single unresponsive API can stall the whole task indefinitely.

**Wrong parameter types** occur when the LLM constructs a tool call with a parameter that doesn't match the schema — a string where an integer is expected, a missing required field, an out-of-range value. These are almost always a description or schema design problem.

**Rate limit failures** happen when tools are called too frequently. A well-designed tool returns explicit rate limit information — *retry after 30 seconds* — so the agent can handle it gracefully rather than hammering the API and making the problem worse.

**Cascading failures** occur in multi-step tasks where step 4 depends on the output of step 3, which failed silently. By step 6 the agent is working from completely wrong premises. This is why silent failures are so dangerous — the damage spreads.

The defense against all of these is the same: design tools to fail explicitly, instrument your executor to catch and report failures, and build retry and fallback logic at the harness level. We go much deeper on failure handling in Part 8.

* * *

## What This Looks Like in Practice

**LangChain** tool definition:

```python
@tool
def search_web(query: str) -> str:
    """Search the web for current information.
    Use for recent facts, news, or data.
    Returns top 5 results as text snippets."""
    return search_api.run(query)
```

**MCP server** tool definition:

```json
{
  "name": "search_web",
  "description": "Search the web for current information...",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": { "type": "string", "description": "Search query" }
    },
    "required": ["query"]
  }
}
```

The logic is identical. The MCP version is portable — it works with any agent that speaks MCP, not just LangChain.

* * *

## Key Takeaways

*   The LLM never executes tools directly — it produces structured text describing an action, the harness runs the actual function, the result comes back as an observation
    
*   Every tool has three parts: name, description, and schema — the description is the most important and most commonly underinvested
    
*   Good tool design means: one job per tool, loud failures, filtered outputs, input validation, and retry-safe operations
    
*   Tools fall into five categories: read, write, execute, communicate, and computer use — each with different risk profiles
    
*   MCP standardizes how agents connect to tools — write the integration once, use it across any MCP-compatible agent or framework
    
*   Silent tool failures are the most dangerous failure mode — they produce wrong answers with full confidence
    
*   Tool design quality directly determines agent reliability — a brilliant LLM with poorly designed tools will underperform a weaker model with well-designed ones
    

* * *

## What's Next

Tools give agents the ability to act. But every action produces a result. Every observation adds to what the agent knows. After enough steps — enough sessions, enough task runs — all of that information has to live somewhere.

And here's the problem most teams discover too late: where that information lives, how it's stored, and how it's retrieved determines whether your agent gets smarter and more reliable over time — or resets to zero on every single run.

That's memory. Four distinct types. Each solving a different problem. And the one most teams get wrong almost guaranteeing production failure.

**Part 5: Memory — How Agents Remember →**

* * *

*Agentic AI Deep Dive — Part 4 of 9*