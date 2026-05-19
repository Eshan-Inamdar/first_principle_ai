---
title: "Agentic AI Deep Dive — Memory: How Agents Remember"
seoTitle: "How AI Agents Remember — The Four Memory Types"
seoDescription: " Learn the four memory types agents
need — in-context, external, episodic, and
procedural — and how each one works."
datePublished: 2026-05-19T17:46:12.807Z
cuid: cmpcxchqb00a72dn8hrqp06b9
slug: agentic-ai-deep-dive-memory-how-agents-remember
cover: https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/656cc365-59ed-4eeb-8c5f-1ed316a3307a.jpg
tags: ai, artificial-intelligence, skills, llm, agents, rag, claude-ai, agentic-ai, agent-memory

---

* * *

## Where We Left Off

In [Part 4](https://eshaninamdar.hashnode.dev/agentic-ai-deep-dive-tools-how-agents-act-on-the-world) we gave agents hands. Tools are how an agent reaches outside itself — into APIs, databases, browsers, file systems. The LLM describes an action, the harness executes it, the result comes back as an observation. MCP standardizes the whole interface so integrations are portable across frameworks.

Every tool call produces a result. Every result is information. After enough steps — enough sessions, enough task runs — that information has to live somewhere.

This part is about where it lives, how it's organized, and why getting it wrong is the most common reason production agents fail.

* * *

## The Hook

Imagine a doctor who wakes up every morning with no memory of any patient they've ever treated.

They're still brilliant. They can diagnose from first principles. They can reason through complex cases. But every patient they see, they start completely from scratch. No case history. No record of what treatments worked or failed. No institutional knowledge built up over years of practice.

They're capable. But they're not reliable. And in medicine — as in production software — reliability is what actually matters.

That's an agent without memory. Capable in a single session. Broken across sessions.

* * *

## The Statelessness Problem

In Part 1, statelessness was one of the four fundamental LLM limitations. We described it abstractly. Now let's make it concrete.

Here is what statelessness looks like in production:

**Problem 1 — No continuity across sessions.** A user asks an agent to help with a project on Monday. On Wednesday they pick up where they left off. The agent has no idea what happened Monday. The user has to re-explain everything. The agent repeats the same suggestions it already made.

**Problem 2 — No learning from past runs.** An agent executes a data pipeline every morning. Monday it makes a mistake — calls the wrong API endpoint. The pipeline fails. Someone fixes it manually. Tuesday the agent makes the exact same mistake. It has no record of Monday.

**Problem 3 — Repeated tool calls.** An agent is halfway through a long task. A step fails and the task restarts. It re-executes every tool call from the beginning — including expensive API calls it already made successfully — because it has no record of what it's already done.

**Problem 4 — Context window overflow.** A long task generates more information than fits in the context window. The agent starts dropping earlier observations. It loses track of decisions it made three steps ago. Its reasoning becomes incoherent.

All four of these are memory problems. And all four have different solutions — because memory in agentic systems is not one thing.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/b0e910e6-5cde-4288-ba2c-b07c09a51d70.png align="center")

* * *

## The Four Types of Agent Memory

We introduced these in Part 2. Now we go deep.

### Type 1: In-Context Memory

**In-context memory** is everything currently inside the LLM's context window — the goal, the conversation history, the tool results so far, any documents passed in.

It's the fastest memory type because no retrieval is needed — it's already there. But it has hard limits.

Every LLM has a maximum context window size — measured in tokens. GPT-4 Turbo supports 128k tokens. Claude supports up to 200k. Gemini goes higher. These sound large until you're running a long multi-step task where every tool result adds thousands of tokens.

When the context fills up, something has to go. Most frameworks use a sliding window — dropping the oldest content. But dropping old observations means the agent can't refer back to decisions it made earlier in the task. This produces the incoherence problem described above.

**What to include in context:** the current goal, recent tool results, the active plan, any constraints. **What to move out:** completed steps, large documents that can be summarized, historical data that can be retrieved on demand.

The discipline of context management — deciding what stays in the window and what moves to external storage — is a core skill in production agent engineering.

### Type 2: External Memory

**External memory** is a persistent store outside the LLM — typically a vector database — that the agent retrieves from on demand.

This is where RAG lives inside an agentic system. Instead of cramming everything into the context window, you store it externally and retrieve only what's relevant to the current step.

How it works:

```plaintext
Information produced → embed as vector → store in vector DB

At retrieval time:
Current query → embed → find nearest vectors → 
retrieve relevant chunks → inject into context
```

The key insight: you don't retrieve everything. You retrieve what's semantically similar to the current question. A vector database finds the most relevant information from potentially millions of stored items and returns only those.

**Common external memory stores:** Pinecone, Weaviate, Chroma, pgvector (Postgres extension). Each has different trade-offs on speed, scale, and cost.

**What goes wrong here:** poor chunking strategy. If documents are split into chunks that are too large, retrieval returns too much noise. Too small, and individual chunks lose context. Chunking strategy is one of the most impactful and least discussed decisions in agent memory design.

### Type 3: Episodic Memory

**Episodic memory** is a record of past interactions and task runs — what the agent did, what worked, what failed.

Think of it as a structured log that the agent can query. Before starting a task, the agent retrieves relevant episodes — *"the last three times I ran this type of pipeline, here's what happened"* — and uses that history to make better decisions.

This is how agents improve over time rather than repeating the same mistakes. Without episodic memory, every task run is the agent's first. With it, the agent builds up institutional knowledge across hundreds of runs.

**Implementation:** episodic memories are typically stored as structured records — timestamp, task type, steps taken, outcomes, errors encountered — in a database the agent can query.

**What goes wrong here:** storing everything without structure. An unstructured log is hard to query usefully. Episodic memory is only valuable if the agent can retrieve *relevant* past episodes, not just recent ones.

### Type 4: Procedural Memory

**Procedural memory** is stored instructions for how to do specific things — reusable processes and skills the agent retrieves and follows.

This is the most practical memory type for most teams because it's the easiest to implement and the most immediately impactful.

CLAUDE.md and SKILL.md are real-world examples of procedural memory in action. Before the agent starts a task, it reads the relevant instruction file — project conventions, available tools, rules to follow, process steps. This is procedural memory retrieved at task initialization.

We go deep on SKILL.md in the next section.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/6646efae-509f-4a4a-a36c-e53ec638e118.png align="center")

* * *

## SKILL.md — Procedural Memory in Practice

SKILL.md is a pattern where you give an agent reusable, retrievable instructions for specific tasks.

The idea is simple. Instead of hoping the agent figures out how to create a PowerPoint presentation correctly every time, you write down exactly how to do it once — which libraries to use, what the output format should be, common pitfalls to avoid — and store it as a skill file. When the agent needs to create a presentation, it retrieves the relevant SKILL.md and follows it.

Here's what a real SKILL.md looks like:

```markdown
# SKILL: Create PowerPoint Presentation

## When to use this skill
When asked to create, generate, or produce a .pptx file.

## Libraries
- Use python-pptx only
- Do NOT use matplotlib for slide generation

## Output path
Always save to /mnt/user-data/outputs/

## Slide structure
- Title slide: title + subtitle only
- Content slides: max 5 bullet points per slide
- Final slide: key takeaways only

## Common mistakes to avoid
- Never use font sizes below 18pt
- Always set slide dimensions to widescreen (13.33 x 7.5 inches)
- Run a validation check after creation
```

Notice what this does. It doesn't make the agent smarter. It makes the agent *consistent*. The same correct process, every time, regardless of which task triggered it.

The difference between CLAUDE.md and SKILL.md:

|  | CLAUDE.md | SKILL.md |
| --- | --- | --- |
| **Scope** | Project-level config | Task-level instructions |
| **When retrieved** | Once, before work begins | When a specific task is triggered |
| **Content** | Project structure, conventions, rules | Step-by-step process for a specific task |
| **Analogy** | Employee onboarding doc | Standard operating procedure |

Both are procedural memory. CLAUDE.md is the onboarding. SKILL.md is the SOP library.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/e0973570-0d17-404d-9808-99d1fdd2c28c.png align="center")

* * *

## How the Memory Types Work Together

No production agent uses just one memory type. They work in combination — each handling a different time horizon and access pattern.

Here's how a well-designed agent uses all four in a single task:

```plaintext
Task begins
     ↓
[Procedural] Read CLAUDE.md and relevant SKILL.md files
     ↓
[Episodic] Retrieve past runs of similar tasks — 
           any failures or patterns to be aware of?
     ↓
Task executes — loop runs
     ↓
[In-context] Current goal, recent tool results, active plan
     ↓
[External] Retrieve relevant documents on demand
           as specific steps require them
     ↓
Task completes
     ↓
[Episodic] Store this run — steps taken, outcomes, 
           any errors encountered
```

The in-context window is the working memory. External is the long-term store. Episodic is the journal. Procedural is the playbook.

Design each layer deliberately. The teams that treat memory as an afterthought — dumping everything into the context window and hoping for the best — are the teams whose agents degrade over time instead of improving.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/9ebd8496-66ca-4370-bf96-417dd0aa536e.png align="center")

* * *

## Vector Memory — How Retrieval Actually Works

External memory deserves a closer look because it's the most technically complex of the four types and the most commonly misunderstood.

The core mechanic is **semantic similarity search**. Instead of searching by keyword, you search by *meaning*.

Every piece of information — a document, a past conversation, a tool result — is converted into a vector: a list of numbers that represents its meaning in high-dimensional space. Similar meanings produce similar vectors. Dissimilar meanings produce vectors that are far apart.

When the agent needs to retrieve information, it converts the current query into a vector and finds the stored vectors that are closest to it. Those closest items are the most semantically relevant — even if they share no keywords.

This is why RAG works for questions like *"what does our refund policy say about international orders?"* even when the policy document uses different words than the question. The meaning is close even if the phrasing isn't.

**The three decisions that determine retrieval quality:**

**Chunking** — how you split documents before embedding. Chunk too large and each vector contains too many concepts, making similarity fuzzy. Chunk too small and individual chunks lose their context. The right chunk size depends on the content type.

**Embedding model** — the model that converts text to vectors. General-purpose models work for general content. Domain-specific content — medical, legal, financial — benefits significantly from domain-specific embedding models.

**Retrieval strategy** — how many chunks to retrieve, whether to rerank results, whether to use hybrid search (combining semantic and keyword). More retrieved chunks means more context but more noise. Getting this balance right is an empirical process.

![](https://cdn.hashnode.com/uploads/covers/6a04980da0c15402774955f3/0daea03b-8a93-4ba7-a278-3edd35b05435.png align="center")

* * *

## Key Takeaways

*   Statelessness produces four concrete production problems: no continuity, no learning, repeated tool calls, and context overflow — each requires a different memory solution
    
*   Memory in agentic systems is four distinct types: in-context, external, episodic, and procedural — not one monolithic system
    
*   In-context memory is fast but limited — context management is a core production skill
    
*   External memory uses vector similarity search to retrieve relevant information on demand — RAG lives here
    
*   Episodic memory is how agents improve over time — structured records of past task runs that the agent can query
    
*   Procedural memory — SKILL.md and CLAUDE.md — is the most immediately practical: reusable instructions that make agents consistent rather than just capable
    
*   Vector retrieval quality depends on three decisions: chunking strategy, embedding model choice, and retrieval configuration — all empirical, all impactful
    
*   Design all four memory layers deliberately — teams that treat memory as an afterthought build agents that degrade rather than improve
    

* * *

## What's Next

One agent with well-designed memory is powerful. It learns. It stays consistent. It doesn't repeat its mistakes.

But some tasks are simply too large for one agent. A task that requires simultaneous research, writing, coding, and client communication can't be handled sequentially by a single agent without becoming slow and brittle. Some tasks need parallelism. Some need deep specialization. Some need one agent to coordinate the work of others.

The answer isn't a smarter single agent. It's a team of agents — each with a specific role, each with its own memory and tools, coordinated by an orchestrator that knows how to delegate.

That's what Part 6 is about.

**Part 6: Multi-Agent Systems — When One Agent Isn't Enough →**

* * *

*Agentic AI Deep Dive — Part 5 of 9*