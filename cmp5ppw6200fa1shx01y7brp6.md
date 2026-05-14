---
title: "Agentic AI Deep Dive — Anatomy of an Agent"
seoTitle: "What's Inside an AI Agent  The Five Core Component Explained"
seoDescription: "Break open the anatomy of an agentic
system — planner, executor, tools, memory, and
the harness that holds it all together."
datePublished: 2026-05-14T16:38:17.886Z
cuid: cmp5ppw6200fa1shx01y7brp6
slug: agentic-ai-deep-dive-anatomy-of-an-agent
cover: https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/8613d65d-54a1-4408-a18a-cbd57e73e55b.jpg
tags: ai, artificial-intelligence, data-science, ai-tools, ai-agents

---

* * *

## Where We Left Off

In [Part 1](https://eshaninamdar.hashnode.dev/agentic-ai-deep-dive-the-problem-with-llms), we established something important. LLMs have four fundamental limitations — knowledge cutoff, no private data, statelessness, and no actions. RAG fixed the knowledge problem. But it didn't give the LLM hands. The breakthrough insight was this: an LLM doesn't have to produce a final answer. It can produce a *next action* — observe what happened — and produce the next action after that. That loop is what makes a system agentic.

Now the obvious question: what exactly is inside that loop?

* * *

## The Hook

Think about what it takes to actually run a professional kitchen.

A brilliant chef isn't enough. You need the equipment, the staff, the suppliers, the order system. Each piece exists for a specific reason. Remove any one of them and the kitchen stops functioning — not because the chef got worse, but because the system around them broke.

An agent is the same idea. The LLM is the intelligence. But intelligence alone doesn't complete tasks. You need the full system — and understanding that system is what separates engineers who can build agents from those who can only use them.

* * *

## If You Were Building an Agent from Scratch

Here is the question that drives this entire part:

> If you had to build something that could take a goal and work through it autonomously — what is the minimum set of things you'd need?

Forget frameworks. Forget libraries. Just think about the problem.

You'd need something to *reason* about the goal. Something to *break it into steps*. Something to *execute those steps*. Something to *interact with the outside world*. And something to *remember* what's happened so far so you don't restart from zero on every action.

That's five things. And those five things map directly to the five core components of every agent ever built — regardless of framework, model, or use case.

We didn't invent these categories. We derived them.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/f9f0417a-3b29-4acf-843d-366a23771df4.png align="center")

* * *

## Component 1: The LLM

The LLM is the reasoning brain of the agent. Every decision flows through it.

It receives the current state of the task — what the goal is, what's happened so far, what tools are available, what the last observation was. And it produces a decision: call this tool with these parameters, or I have enough information to produce a final response.

This is the most misunderstood component. People treat the LLM as the whole agent. It isn't. It's the decision-maker inside the agent. Everything else exists to give it good information and carry out its decisions.

What goes wrong here: when an agent fails, the instinct is to blame the model — use a better model, re-prompt it differently. Often the real failure is in the components surrounding it. The model made a reasonable decision. The rest of the system let it down. Always check the other components before changing the model.

* * *

## Component 2: The Planner

The **planner** is how an agent breaks a complex goal into a sequence of steps.

Given a goal like *"research our top five competitors, extract their pricing, and draft a comparison report"* — the planner breaks that into discrete steps:

1.  Search for competitor 1
    
2.  Extract pricing from their site
    
3.  Repeat for competitors 2 through 5
    
4.  Synthesize findings
    
5.  Draft the report
    

Not every agent has an explicit separate planner. In simpler agents, the LLM plans inline as part of its reasoning — thinking through the next step each time around the loop. In more complex agents handling long multi-step tasks, the planner is a distinct component that maintains a persistent plan even as individual steps fail and get retried.

**The trade-off:** explicit planners make agents more reliable on long tasks but add architectural complexity. For short tasks, inline planning is usually sufficient. Know which one you need before you build.

* * *

## Component 3: The Executor

The **executor** is what actually *runs* the actions the LLM decides on.

The LLM outputs a decision — *"call the search API with query X"*. The executor handles the mechanics: formatting the API call, managing timeouts, catching errors, returning the result back to the LLM as an observation.

This separation matters. You can change which tools are available, add error-handling logic, or swap out the executor entirely — without touching the LLM or the planner. Clean boundaries make systems maintainable.

**What goes wrong here:** most agent failures that *look* like reasoning failures are actually executor failures. The LLM made a sound decision. The executor failed silently — hit a timeout, returned a malformed result, swallowed an error — and the LLM had nothing good to reason from on the next step. Always instrument your executor before you blame your model.

* * *

## Component 4: Tools

**Tools** are how the agent interacts with the world outside itself.

A tool is any capability the agent can invoke:

*   A web search API
    
*   A database query function
    
*   A code interpreter
    
*   A file read/write operation
    
*   A browser automation function
    
*   An email or calendar API
    

Each tool has three things: a **name**, a **description** so the LLM knows when to use it, and a **schema** so the LLM knows what parameters to pass.

The LLM doesn't know how any tool works internally. It just knows: *"I can call this tool with these inputs and get back a result."* This is intentional. The LLM reasons about *what* to do. The tool handles *how* to do it.

**This is why tool design is an underrated skill.** A well-designed tool with a clear, precise description gets used correctly. A vague description results in wrong parameters — or the tool not being called at all when it should be. We go deep on tool design, and on MCP (the emerging standard for how agents connect to tools), in Part 4.

* * *

## Component 5: Memory

**Memory** is what allows the agent to maintain context, learn from what's happened, and not restart from zero on every action.

There are four distinct types — and understanding the difference between them is important:

**In-context memory** is what's currently inside the LLM's context window — the goal, tool results, conversation so far. Fast and immediate, but strictly limited in size and wiped when the session ends.

**External memory** is a vector database or document store the agent retrieves from at query time. Persistent and large — can hold millions of documents — but requires an explicit retrieval step. This is where RAG lives inside an agentic system.

**Episodic memory** is records of past interactions and task runs — what happened the last time this agent executed a similar task. It's what lets an agent improve over time rather than repeating the same mistakes.

**Procedural memory** is stored instructions for how to do specific things. Skills and processes the agent retrieves and follows. CLAUDE.md and SKILL.md files are real-world examples of this — more on them shortly.

Memory gets its own full part — Part 5 — because it's where most production agents break. For now, understand that memory is not one thing. It's a stack of four distinct systems, each solving a different problem.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/4868b5a9-efed-42e5-8d8d-40ad04d7a07d.png align="center")

* * *

## The Agent Harness

We have five components. But we haven't described what holds them together.

The **agent harness** — sometimes called the runtime or orchestration layer — is the chassis that:

*   Manages the agentic loop
    
*   Routes the LLM's decisions to the executor
    
*   Passes tool results back to the LLM as observations
    
*   Handles errors and retries
    
*   Decides when the loop ends
    

The harness is invisible when it works. It becomes the most painful thing to debug when it doesn't.

This is what frameworks like **LangChain**, **LangGraph**, and **CrewAI** actually are at their core. Not magic. Just well-engineered harnesses that manage the loop so you don't have to build the plumbing yourself.

| Framework | Type | Best for |
| --- | --- | --- |
| **LangChain** | General purpose | Most single-agent tasks |
| **LangGraph** | Graph / state machine | Complex flows needing precise loop control |
| **CrewAI** | Multi-agent | Teams of specialized agents — Part 6 territory |

Choosing the right harness is an architectural decision. We'll revisit this in Part 6.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/98b8bb75-f863-4dc9-a8de-4cf2c07cbdb7.png align="center")

* * *

## CLAUDE.md — Briefing the Agent Before It Starts

Before the loop begins, an agent needs context. What environment is it working in? What rules apply? What tools are available and how should they be used?

**CLAUDE.md** is a file you place in your codebase when using Claude Code — Anthropic's agentic coding tool. Before the agent starts working, it reads this file. It functions as a pre-task configuration layer — not part of the loop, but essential setup before the loop begins.

Here's what a real CLAUDE.md looks like:

```markdown
# Project: Customer Analytics Dashboard

## Stack
- Python 3.11, FastAPI backend
- React + TypeScript frontend
- PostgreSQL database

## Conventions
- Always write tests for new functions
- Use snake_case for Python, camelCase for TypeScript
- Never modify the /legacy folder without asking first

## Available tools
- run_tests: runs the full test suite
- query_db: read-only database access
- deploy_staging: deploys to staging environment

## Rules
- Always run tests before suggesting a PR
- If unsure about a schema change, ask before proceeding
```

Notice what this does. It doesn't change the model. It doesn't retrain anything. It gives the agent the context it needs to work correctly from the first step — project structure, conventions, constraints, available tools.

This is **procedural memory** in action. The agent doesn't figure out your project conventions from scratch every time. You hand it the configuration. It reads it. Work begins with the right foundations already in place.

SKILL.md files work the same way — reusable instruction sets for specific tasks that an agent retrieves before executing. We'll cover both in depth in Part 5.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/3d7f8964-7b30-4e38-830b-424c1c39ced9.png align="center")

* * *

## How the Components Connect

Here is the full flow from goal to completion:

```plaintext
Goal arrives
     ↓
Planner breaks it into steps
     ↓
LLM reasons about the current step
     ↓
LLM decides: call a tool
     ↓
Executor runs the tool
     ↓
Result returned as observation
     ↓
LLM reasons about the observation
     ↓
Repeat until goal is complete
     ↓
Final response produced
```

The harness manages this entire loop. The LLM never touches the tool directly. The tool never talks to the planner. Every component has one job and communicates only with what it needs to.

This is not accidental. Clean separation of components is what makes agents debuggable. When something goes wrong, you can isolate exactly which component failed — the reasoning, the planning, the execution, or the tool itself. That isolation is only possible because the boundaries are clear.

* * *

## What This Looks Like in Practice

**Task:** *"Monitor our competitor's pricing page every morning, identify any changes, and send a Slack summary if anything changed."*

| Step | Component | What happens |
| --- | --- | --- |
| Receive task | Harness | Loop initialized |
| Break into steps | Planner | Fetch → compare → summarize → notify |
| Fetch pricing page | Tool | HTTP tool retrieves the page |
| Compare to yesterday | Memory | External memory retrieves yesterday's snapshot |
| Reason about differences | LLM | Decides if changes are significant |
| Draft summary | LLM | Writes the Slack message content |
| Send Slack | Tool | Slack API tool sends the message |
| Store today's snapshot | Memory | External memory saves current version |
| Exit | Harness | Task complete, loop ends |

Eight steps. Five components. One harness. Nothing exotic — just well-orchestrated simple parts, each doing one job cleanly.

* * *

## Key Takeaways

*   Every agent has five core components: LLM, planner, executor, tools, and memory — regardless of framework or use case
    
*   The LLM is the decision-maker inside the agent, not the whole system — check surrounding components before blaming the model
    
*   The agent harness manages the loop — LangChain, LangGraph, and CrewAI are implementations of this idea
    
*   Tools need names, descriptions, and schemas — tool design quality directly affects agent reliability
    
*   Memory is four distinct systems: in-context, external, episodic, and procedural — each solving a different problem
    
*   CLAUDE.md is procedural memory in practice — pre-task configuration that gives an agent the right context before the loop begins
    
*   Clean component separation is what makes agents debuggable in production
    

* * *

## What's Next

We now know what's inside an agent. Every component, every boundary, every responsibility.

But knowing the components isn't the same as understanding how an agent *thinks*. Given a goal with no clear path, incomplete information, and tools that sometimes fail — how does it decide what to do next? How does it recover when a step doesn't work? How does it know when it's actually done?

That's the reasoning loop. And it's where the real intelligence of an agentic system lives.

**Part 3: How Agents Think — Reasoning & Planning Loops →**

* * *

*Agentic AI Deep Dive — Part 2 of 9*